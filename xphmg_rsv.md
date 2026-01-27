# RISC-V Scalar-Vectors (XPHMG_RSV)

**Category:** Vendor Extension (`XPHMG_RSV`)
**Version:** 0.1.1
**Author:** Pedro H. M. Garcia
**Based on concepts by:** Luke Kenneth Casson Leighton (*Libre-SOC / Simple-V*)
**License:** CC-BY-SA 4.0 (open specification)

> This document describes an experimental vendor extension of the RISC-V ISA that adapts the *Simple-V* (SVP64) model created by Luke Kenneth Casson Leighton for Libre-SOC.
> The purpose is to enable lightweight vectorization by reinterpretation of existing scalar instructions while remaining binary-compatible with standard RISC-V cores when disabled.

---

## 1  Overview (Informative)

`XPHMG_RSV` (RISC-V **Scalar-Vectors**) defines a lightweight **scalar-vector reinterpretation layer** for the RISC-V ISA. Rather than introducing a separate vector register file or a distinct execution model, RSV reuses existing scalar instructions and applies them repeatedly across a programmable register window under explicit control of vector length, stride, and predicate masks (XPHMG_PMASK). RSV is designed as a *structural adaptation* of scalar code, not as a replacement for the standard RISC-V Vector (V) extension; when disabled, all instructions execute with their original scalar semantics and remain fully compatible with unextended RISC-V cores.

RSV vector operations, when executed under a common predication context and synchronized using XPHMG-defined barriers, may also be used to express subgroup-style collective behavior. Such usage does not introduce new architectural semantics and remains valid on pure SIMD implementations.

Unlike classical vector ISAs, RSV does not define its own numeric or memory policy. All precision, rounding, saturation, exception handling, masking semantics, and gather/scatter behavior are **architecturally inherited** from `XPHMG_CAP` and `XPHMG_XMEM`. At each instruction boundary, an RSV engine snapshots the effective `CAP.PREC.*` and `CAP.XMEM.*` state, ensuring consistent behavior across RSV, MTX, RT, GFX, and ACCEL domains.

### 1.1 Conceptual Model (Informative)

* RSV acts as a per-instruction micro-loop: a scalar instruction is reissued for `VL` iterations using base/stride parameters taken from SV CSRs, with predicate gating of lane execution and writeback via XPHMG_PMASK.
* No additional architectural register file is introduced; RSV iterates over slices of the existing scalar register file according to `SVSRCA/SVSRCB/SVDST` base+stride state.
* Underlying scalar instruction semantics (ALU, FP, load/store, CSR side effects) are preserved per lane; RSV only replicates the execution across lanes defined by `VL` and strides.
* When inactive (`SVSTATE.EN=0`), RSV introduces no additional architectural effects beyond those defined by the base ISA and enabled extensions.
* Execution parameters or control data for RSV kernels may, in some implementations, be sourced from memory-resident data structures produced by prior execution phases using RSV and XMEM.

### 1.2 Architectural Guarantees & Invariants (Informative)

* Numeric behavior (rounding, saturation, NaN handling, quantization, SAE/ZMODE defaults) is inherited from `CAP.PREC.*` with no RSV-local variants.
* Memory behavior (coherence domain, streaming profiles, gather/scatter addressing) is inherited from `CAP.XMEM.*`; RSV does not define independent memory policies.
* Effective CAP and XMEM state is latched at each RSV-affected instruction boundary; per-instruction overrides (`svon.fpctl`) apply only to the immediately following instruction and do not change global CAP state.
* Masked writeback follows `CAP.PREC.MODE.ZMODE`; predicated-off lanes either merge or zero according to the effective CAP setting.
* Lane 0 behavior under RSV matches scalar execution of the same instruction with identical operand values; disabling RSV yields results bit-equivalent to scalar execution.
* Unsupported encodings or element widths continue to raise the same architectural exceptions as the underlying scalar instruction (e.g., *Illegal Instruction*), unless otherwise specified in later sections.

### 1.3 Interaction with CAP / XMEM / Other XPHMG Extensions (Informative)

* CAP supplies all precision, exception, and saturation policy consumed by RSV; sticky bits and exception enables in CAP govern RSV lane outcomes.
* XMEM supplies coherence, streaming, compression, and gather/scatter configuration; RSV reuses SVMEM settings and defaults defined by XMEM.
* MTX, RT, GFX, and ACCEL domains may produce or consume data transformed by RSV; interoperability is maintained by the shared CAP/XMEM contract and by the requirement that RSV snapshots effective CAP/XMEM state at instruction boundaries.
* Coexistence with the RISC-V V extension is supported; RSV does not alias V state and remains orthogonal to `v*` register files and VLMAX semantics.

### 1.4 Undefined / Reserved Behavior (Informative)

* Encodings, CSR fields, or modes not defined in this specification remain reserved and should trap as *Illegal Instruction* or behave as defined by the base ISA and CAP/XMEM defaults.
* Behavior of vendor-specific CAP or XMEM extensions not covered here is outside the scope of RSV v0.1.

### 1.5 Notes for Implementers (Informative)

* RSV can be realized as a decode-time micro-loop or similar mechanism; the micro-architectural strategy is implementation-defined provided the architectural invariants above are preserved.
* Minimal implementations may omit optional saturation CSRs as described later; omitted CSRs must behave as specified in the Architectural State section.
* Toolchains should assume RSV-enabled binaries remain executable on cores where RSV is disabled, with RSV prefixes treated as *Illegal Instruction* unless trapped or emulated by higher privilege firmware.

---

## 2  Scope, Goals, and Relationship to CAP/XMEM (Informative)

### 2.1 Scope Boundaries

* RSV provides scalar-vector reinterpretation of existing scalar instructions using base+stride state held in SV CSRs and predicate state sourced from XPHMG_PMASK.
* RSV does not introduce new arithmetic semantics, new register files, or new memory-side policies; it reuses existing scalar execution units and register files.
* RSV operates independently of the RISC-V Vector (V) extension; neither reuses the other's architectural state.

### 2.2 Goals

* Enable lightweight vectorization of scalar code sequences with minimal encoding overhead and without duplicating functional units.
* Ensure that when RSV is inactive, scalar instruction semantics are observed without additional architectural side effects.
* Ensure numeric and memory outcomes match those defined by CAP and XMEM so that data can flow consistently across RSV, MTX, RT, GFX, ACCEL, and scalar domains.
* Allow incremental hardware adoption by keeping required architectural state limited to the SV CSR block.

### 2.3 Non-Goals

* RSV is not intended to replace the RISC-V Vector (V) extension or to emulate VLMAX-based semantics.
* RSV does not define new numeric modes, memory coherency domains, or exception models beyond what CAP and XMEM already provide.
* RSV does not guarantee latency or throughput characteristics; microarchitectural realization is implementation-defined.

### 2.4 Relationship to CAP (Precision, Exceptions, Numeric Policy)

* RSV inherits all numeric policy (precision selection, rounding, NaN handling, saturation, quantization, SAE defaults, ZMODE) from `CAP.PREC.*`.
* RSV lanes update CAP sticky status (`CAP.PREC.STAT`, `CAP.PREC.EXC.ST`) in the same manner as scalar execution.
* Per-instruction overrides via `svon.fpctl` are applied only to the next RSV-affected instruction and do not modify the underlying CAP state.
* There are no RSV-local extensions or variants of CAP precision behavior.

### 2.5 Relationship to XMEM (Memory, Coherency, Streaming, Gather/Scatter)

* RSV reuses XMEM defaults for coherence (`CAP.XMEM.DOM`), memory class mapping, streaming profiles, compression, and descriptor policies.
* RSV gather/scatter forms use SVMEM configuration (`SVMBASE/SVMIDX/SVMSCL/SVMLEN/SVMMODE`) as defined by XMEM; masked-off lanes must not issue memory requests.
* RSV may not define independent memory policies; deviations from XMEM behavior are reserved.

### 2.6 Relationship to Other XPHMG Extensions

* MTX, RT, GFX, and ACCEL consume or produce data that must remain consistent with CAP/XMEM policy; RSV relies on the shared CAP/XMEM contract to avoid per-domain reinterpretation.
* Tile- or tensor-oriented layouts produced by MTX are consumable by RSV when layout metadata indicates compatibility (see Cross-Domain Interoperability).
* Scalar ALU execution remains the reference for RSV lane 0; disabling RSV yields scalar-equivalent results.

### 2.7 Notes for Implementers (Informative)

* Implementations may pipeline or micro-loop RSV behavior at decode or issue, provided CAP/XMEM state is latched per instruction boundary as required elsewhere in this specification.
* Minimal implementations that omit optional SV CSRs (e.g., saturation overrides) must honor the defined reset and read/write behaviors in the Architectural State section.
* Toolchains may assume RSV code remains executable on non-RSV cores via traps or emulation; no RSV-specific relocation or ABI changes are required.

NOTE: Any vendor-specific CAP or XMEM extensions not described here are outside the scope of RSV v0.1; their interaction with RSV may be clarified in a future revision.

---

## 3  Conformance and Compatibility (Normative)

### 3.1 Baseline ISA Compatibility

* Conforming implementations support **RV32I** or **RV64I** and may include any standard extension.
* When RSV is inactive (`SVSTATE.EN=0`), instructions execute with their original scalar semantics; no RSV side effects are observable.
* RSV uses only vendor-defined opcode space (`custom-0` for prefixes, `custom-1` for optional profiles). Systems that do not implement RSV may treat these opcodes as *Illegal Instruction* per base ISA rules.

### 3.2 Coexistence with Other Extensions

* RSV coexists with the RISC-V Vector (V) extension; RSV state does not alias the `v*` register file or VLMAX semantics.
* RSV coexists with other XPHMG extensions (CAP, XMEM, MTX, RT, GFX, ACCEL) through the shared CAP/XMEM contract; enabling RSV must not alter their architectural state except through defined CAP/XMEM mechanisms.

### 3.3 CSR Allocation and Access

* RSV CSRs occupy the vendor range **0x7F8-0x7FF**. Accesses to unimplemented RSV CSRs must trap as *Illegal Instruction*.
* Optional CSRs (e.g., saturation override) obey the read/write behaviors defined in the Architectural State section; absence of an optional CSR does not change the numbering of other RSV CSRs.

### 3.4 Dependency on CAP/XMEM Semantics

1. RSV gather/scatter must follow SVMEM rules from XMEM; RSV may not define independent gather/scatter behavior.
2. Effective predication from PMASK (selected by pbank) gates writeback; `CAP.PREC.MODE.ZMODE` selects merge vs zero for predicate-false lanes.
3. Floating-point exceptions and precision behavior follow CAP global semantics; RSV does not define its own exception model.

---

## 4  Programmer's Model (Normative)

### 4.1 Conceptual Loop

When XRSV is active, the processor repeats a scalar operation `OP rd, rs1, rs2` `VL` times:

```
for i = 0 .. VL-1:
    if pred[i]:
        rd[baseD + i*incD] = OP(rs1[baseA + i*incA], rs2[baseB + i*incB])
```

| Field        | Meaning                              |
| ------------ | ------------------------------------ |
| **VL**       | Vector length (number of iterations) |
| **incA/B/D** | Register increments (stride)         |
| **pred[i]**  | Predicate bit from PMASK bank `pbank_eff` |

`inc=0` => broadcast. VL >= 1.

### 4.2 Register Windowing and Strides

* Base registers and strides for sources and destination are taken from `SVSRCA`, `SVSRCB`, and `SVDST` at decode time.
* Stride values select element spacing within the scalar register file; `inc=0` broadcasts a single element, positive increments advance through consecutive registers.
* Implementations must apply strides consistently for all lanes of a given RSV-affected instruction.

### 4.3 Predicate Masking

* RSV predication consumes only the **effective predication mask** defined by `XPHMG_PMASK`.
* For each RSV-affected instruction, an effective PMASK bank `pbank_eff` is selected as follows:
  1. If the instruction encoding explicitly carries a `pbank` field (implementation-defined per instruction family), that value is used.
  2. Otherwise, `SVSTATE.PBANK` is used.
  3. If neither is implemented, `pbank_eff` defaults to `0`.
* `pbank=0` refers to **PMASK bank0**, which is **architecturally virtual ALL-ONES** (all lanes active), regardless of physical storage.
* For lane `i`, `pred[i] = PMASK.bank[pbank_eff][i]` as defined by `XPHMG_PMASK`.
* **Masked-off lanes MUST NOT execute**: they MUST NOT read operands, MUST NOT generate exceptions, and MUST NOT produce architecturally visible side-effects.
* For any instruction form that would otherwise perform memory transactions (including loads, stores, AMOs/atomics, and gather/scatter), **masked-off lanes MUST NOT emit memory accesses**. This requirement is satisfied by consuming XMEM semantics without issuing per-lane requests for `pred[i]=0`.
* Destination writeback for masked-off lanes is governed exclusively by `CAP.PREC.MODE.ZMODE` (merge vs zero) (see Section 9).

### 4.4 Vector Length (VL)

* VL is established via `svsetvl` or `svp.one.vlstep` and latched in `SVSTATE`.
* VL >= 1; VL affects the number of loop iterations for RSV-affected instructions until SV state is changed or expires (ONE_SHOT/BLK semantics in Section 7).
* Implementations may clamp requested VL to an implementation-defined maximum; the effective VL is reported by `svsetvl`.

### 4.5 Interaction with Scalar Semantics

* The scalar instruction executed under RSV retains its architectural meaning; RSV supplies only repetition, base/stride, and predication context.
* Side effects of the scalar instruction (e.g., CSR accesses, flags) apply per lane unless the underlying instruction specifies otherwise.
* Lane 0 behavior under RSV is architecturally identical to executing the scalar instruction with the same operands when RSV is disabled.

### 4.6 Interaction with CAP/XMEM State

* Effective precision, rounding, saturation, SAE, ZMODE, and memory defaults are taken from CAP/XMEM snapshots at instruction boundary (see Section 9).
* RSV does not redefine address generation or memory ordering; memory-touching scalar instructions under RSV use XMEM-defined SVMEM settings when applicable.

### 4.7 Notes for Implementers (Informative)

* The micro-loop realization (e.g., unrolled, pipelined, or sequenced) is implementation-defined, provided architectural ordering and CAP/XMEM latching rules are respected.
* Predicate evaluation and stride application occur per lane; implementations may short-circuit masked lanes to avoid unnecessary computation or memory access, while still honoring ZMODE for masked writeback.

---

## 5  Architectural State (Normative)

### 5.1 CSR Map (0x7F8 - 0x7FF)

| Addr  | Name        | Purpose                                                                 |
| ----- | ----------- | ----------------------------------------------------------------------- |
| 0x7F8 | **SVSTATE** | Enable and lifetime control; VL; mode flags; **PBANK** selection.       |
| 0x7F9 | **SVSRCA**  | Source A base + stride.                                                 |
| 0x7FA | **SVSRCB**  | Source B base + stride.                                                 |
| 0x7FB | **SVDST**   | Destination base + stride.                                              |
| 0x7FC | *(reserved)*| Reserved (must trap if accessed).                                       |
| 0x7FD | *(reserved)*| Reserved (must trap if accessed).                                       |
| 0x7FE | **SVSAT**   | Saturation / element-width override (optional).                         |
| 0x7FF | **SVFAULTI**| Index of element that faulted.                                          |

### 5.2 CSR Definitions

* **SVSTATE (0x7F8)**: Holds enable (`EN`), one-shot/block control (`ONE_SHOT`, `BLK`), latched VL, mode flags, and the active PMASK bank selector **`PBANK`**. `PBANK` selects the `XPHMG_PMASK` bank used for effective predication when the instruction form does not provide an explicit `pbank` field. `PBANK` is sampled at decode for each RSV-affected instruction. BLK decrements as described in Section 7; when BLK reaches zero, `EN` clears. Reset value is 0 (implying `PBANK=0`).
  * `PBANK` (WARL): Optional PMASK bank select consumed by RSV predication. If unimplemented, `PBANK` reads as 0.
* **SVSRCA / SVSRCB / SVDST (0x7F9-0x7FB)**: Provide base register indices and strides for source A, source B, and destination. Base and stride fields are sampled at decode and applied uniformly across all lanes of the instruction. Reserved bits read as zero; writes to reserved bits are ignored. Reset values are 0.
* **SVSAT (0x7FE, optional)**: Provides saturation and element-width override controls where implemented. If unimplemented, reads return zero and writes are ignored; saturation events must still update CAP precision status as defined in Section 9.
* **SVFAULTI (0x7FF)**: Records the index of the first faulting element for an RSV-affected instruction. Reset value is 0. If no fault has occurred, the contents are implementation-defined. NOTE: Write semantics for clearing or setting SVFAULTI are implementation-defined in v0.1 and may be clarified in a future revision.

### 5.3 Access, Reset, and Privilege

* All RSV CSRs are Machine-mode read/write; accesses to unimplemented RSV CSRs must raise *Illegal Instruction*.
* Reset values are as specified above; optional CSRs that are not implemented must still preserve the register numbering.
* CSR writes while RSV is active take effect on subsequent RSV-affected instructions; the current instruction uses the CSR snapshot taken at decode.
* Reserved fields within RSV CSRs read as zero and ignore writes.

### 5.4 Architectural Invariants

* SV CSR state used by an instruction is the snapshot taken at decode; later CSR changes do not affect the instruction in flight.
* Optional CSRs that read as zero (e.g., saturation override) must not alter the behavior of implemented CSRs or numbering.
* VL in SVSTATE governs repetition until explicitly changed or until the block counter expires (see Section 7).

### 5.5 Notes for Implementers (Informative)

* SV CSRs are XLEN-wide and follow standard SYSTEM CSR access rules. Implementations may implement WARL behavior on unsupported fields provided architectural effects remain consistent with the definitions above.
* Microarchitectures may cache or shadow SV CSR fields for efficiency, but architectural visibility must reflect the rules above for reads, writes, and decode-time sampling.

---

## 6  Instruction Set Overview (Normative)

### 6.1 Prefix Instruction Encodings (custom-0 opcode = `0001011`)

| imm[11:0]      | rs1     | funct3 | rd      | opcode    | Mnemonic             | Description                                                                       |
| -------------- | ------- | ------ | ------- | --------- | -------------------- | --------------------------------------------------------------------------------- |
| `000000000000` | `xxxxx` | 000    | `xxxxx` | `0001011` | **svsetvl rd, rs1**  | Set VL = `rs1[7:0]`; return effective VL in `rd`.                                 |
| `0000xxxxxxxx` | `00000` | 000    | `xxxxx` | `0001011` | **svsetvl rd, imm8** | Immediate VL (1-256).                                                             |
| `000000000001` | `00000` | 001    | `00000` | `0001011` | **svon.one**         | Enable RSV for next instruction (ONE_SHOT).                                       |
| `0000xxxxxxxx` | `00000` | 010    | `00000` | `0001011` | **svon.blk imm8**    | Enable for N instructions (`BLK=N`).                                              |
| `000000000000` | `00000` | 011    | `00000` | `0001011` | **svend**            | Disable RSV immediately.                                                          |
| `vvvvvvbbbddd` | `00000` | 100    | `00000` | `0001011` | **svp.one.vlstep**   | Compact one-shot: `VL=(imm[11:6]+1)`, base step=`imm[5:3]`, dest step=`imm[2:0]`. |
| `0000000zzzzz` | `00000` | 101    | `00000` | `0001011` | **svon.fpctl**       | Overrides floating-point control for the **next** instruction: rounding (`rc`), exception suppression (`sae`), and mask-write mode (`z`). |
> If an unsupported RC is requested, hardware must use CAP.PREC.MODE.FP_RMODE and may set a soft status in CAP.PREC.STAT (implementation-defined).

Step codes (`bbb` / `ddd`):

| Bits | Meaning              |
| ---- | -------------------- |
| 000  | broadcast (stride 0) |
| 001  | +1                   |
| 010  | +2                   |
| 011  | +4                   |
| 1xx  | reserved             |

zzzzz: `imm[4:0] = rc[2:0], sae, z`
```
imm[4:2] -> RC (0=RNE,1=RTZ,2=RDN,3=RUP,4=RMM)
imm[1]   -> SAE (1 = suppress exceptions)
imm[0]   -> ZMODE (1 = zeroing, 0 = merge)
```

### 6.2 Operational Semantics for Prefixes

* Prefixes execute in the scalar pipeline and program SVSTATE and related CSRs used by the immediately following (or next N) RSV-affected instructions.
* `svsetvl` writes VL into SVSTATE and returns the effective VL in `rd`; immediate and register forms are provided. Implementations may clamp VL to a maximum; the reported VL reflects the effective value.
* `svon.one` sets `SVSTATE.EN=1, ONE_SHOT=1`; `svon.blk` sets `EN=1, ONE_SHOT=0, BLK=N`; `svend` clears `EN` immediately. BLK countdown behavior is defined in Section 7.
* `svp.one.vlstep` programs a one-shot with compact encoding of VL and base/dest stride steps using the step code table above.
* `svon.fpctl` applies floating-point control overrides (RC, SAE, ZMODE) to the immediately following instruction only; if omitted, defaults are taken from `CAP.PREC.MODE.{FP_RMODE,SAE_DEF,ZMODE}`. Overrides do not modify CAP state and expire after one instruction.

### 6.3 Architectural Invariants

* Prefixes do not directly perform lane-wise computation; they configure SVSTATE and related CSRs for subsequent RSV-affected instructions.
* All prefix effects are sampled at decode of the instruction they target; later CSR writes do not retroactively alter an instruction already decoded.
* Unsupported encodings must trap as *Illegal Instruction* in accordance with base ISA rules; reserved fields and step codes must not introduce new semantics in v0.1.

### 6.4 Interaction with CAP/XMEM and Other Extensions

* `svon.fpctl` consumes CAP precision controls but does not update CAP state; ZMODE overrides from fpctl apply to mask-off behavior for the next instruction only.
* Memory-related behavior of subsequent RSV-affected instructions remains governed by XMEM (SVMEM, coherence, streaming) as described in Sections 4 and 9; prefixes do not alter XMEM state.
* Coexistence with the RISC-V V extension is unchanged; RSV prefixes occupy custom-0 and do not alias V-encoded instructions.

### 6.5 Notes for Implementers (Informative)

* Prefixes may be decoded early to preload SVSTATE without stalling the pipeline; care must be taken to honor instruction-boundary latching rules.
* `svon.fpctl` applies to exactly one instruction; implementations may clear the override automatically after consumption to avoid state leakage.
* Illegal or reserved combinations should trap rather than silently degrade to maintain debuggability and compliance.

---

## 7  Execution Model (Normative)

### 7.1 Enabling and Lifetime (ONE_SHOT / BLK)

* `svon.one` sets `SVSTATE.EN=1, ONE_SHOT=1`; the immediately following instruction observes RSV state and then clears `EN`.
* `svon.blk N` sets `EN=1, ONE_SHOT=0, BLK=N`; each RSV-affected instruction decrements `BLK` after completion. When `BLK` reaches zero, `EN` clears automatically.
* `svend` forces `EN=0` immediately, regardless of BLK. Subsequent instructions execute as scalar until RSV is re-enabled.
* If RSV is disabled (EN=0) at decode, the instruction executes with scalar semantics and does not consume RSV CSRs.

### 7.2 CSR Snapshot Timing

* At instruction decode, the implementation snapshots SVSTATE, SVSRCA, SVSRCB, SVDST, SVSAT, and relevant CAP/XMEM state (see Section 9) for the RSV-affected instruction.
* Changes to SV CSRs or CAP/XMEM after decode do not affect the instruction already decoded.
* If BLK expires or EN clears after decode but before completion, the in-flight instruction completes with the snapshot taken at decode.

### 7.3 Loop Execution and Stride Application

* For each lane `i` in `0..VL-1`, source and destination register indices are computed from the decoded base and stride fields in SVSRCA/SVSRCB/SVDST.
* Strides apply uniformly across all lanes of the instruction; stride or base misalignment is not corrected by the implementation and may lead to overlapping registers if programmed so by software.
* Predicate bits from PMASK (per `pbank_eff`) gate lane execution and writeback; masked-off lanes follow `CAP.PREC.MODE.ZMODE` for destination writeback.

### 7.4 Interaction with CAP/XMEM State

* Numeric behavior (precision, rounding, saturation, SAE, ZMODE) is governed by the latched CAP state (and any per-instruction `svon.fpctl` override) at decode.
* Memory behavior for any load/store executed under RSV follows the latched XMEM state, including SVMEM parameters. Masked-off lanes must not issue memory requests.

### 7.5 Faults and Exceptions

* Fault handling is described in Section 8. The first faulting lane records its index in SVFAULTI and, if enabled, raises the corresponding trap.
* SAE overrides from `svon.fpctl` suppress traps but still update sticky status bits in CAP.

### 7.6 Notes for Implementers (Informative)

* Implementations may pipeline or unroll the micro-loop but must maintain per-instruction atomicity with respect to the latched SV and CAP/XMEM state.
* BLK decrement and EN clearing occur after the RSV-affected instruction completes; partial execution must not leave SVSTATE in an indeterminate state.
* Overlapping register windows programmed by software are permitted architecturally; hardware need not detect or prevent overlap.

---

## 8  Exception and Fault Model (Normative)

### 8.1 Fault Recording and Trap Model

* The first faulting lane index is written to `SVFAULTI` when an RSV-affected instruction encounters a fault; subsequent lane faults do not overwrite `SVFAULTI` for that instruction.
* Faults are delivered as traps consistent with the underlying scalar instruction semantics, subject to CAP exception enables and SAE controls.
* If RSV is disabled (EN=0), `SVFAULTI` is not modified by scalar execution of the instruction.

### 8.2 Floating-Point Exceptions (CAP Integration)

1. If `svon.fpctl.sae=1` or `CAP.PREC.MODE.SAE_DEF=1`, all synchronous FP traps are suppressed; sticky bits in `CAP.PREC.EXC.ST` are still updated.
2. If SAE is not in effect and `CAP.PREC.EXC.EN` enables the raised condition (`NV,DZ,OF,UF,NX`), a trap is delivered after recording the first faulting lane.
3. Implementations must update `CAP.PREC.STAT.EFF_SAE` to reflect the effective SAE mode active during the faulting instruction.
4. Only the first faulting lane produces an architecturally visible trap; sticky bits reflect all lanes that encounter exceptions.

### 8.3 Integer and Memory Faults

* Integer faults (e.g., divide-by-zero in integer ops) and memory faults (e.g., access faults, page faults) are reported according to the underlying scalar instruction semantics.
* Masked-off lanes must not generate memory requests; therefore, they cannot generate memory faults.
* Memory ordering and coherence follow XMEM; traps raised due to memory faults occur as if the scalar instruction executed once, with `SVFAULTI` capturing the first faulting lane.

### 8.4 Interaction with Predication and ZMODE

* Predicated-off lanes do not raise traps and do not update `SVFAULTI`.
* ZMODE controls only masked writeback behavior and does not affect exception detection; exception detection is based on the computation or memory access for active lanes.

### 8.5 Undefined or Reserved Behavior

* Behavior for unimplemented or reserved RSV encodings follows the base ISA *Illegal Instruction* trap model.
* If `SVFAULTI` is read before any fault has occurred, its value is implementation-defined.

### 8.6 Notes for Implementers (Informative)

* Implementations may pipeline lane execution; the architectural requirement is that only the first faulting lane updates `SVFAULTI` and can trigger a trap (if enabled), with sticky bits reflecting all affected lanes.
* SAE suppression should not prevent sticky flag updates in CAP; ensure that suppression does not clear or mask architecturally required status bits.
* Fault replay or debug mechanisms should preserve the latched CAP/XMEM state and the recorded `SVFAULTI` to enable deterministic re-execution.

---

## 9  Architectural Interaction with CAP/XMEM (Normative)

### 9.1 Precision and Exception Control (CAP Integration)

* RSV interprets scalar FP and integer instructions according to `CAP.PREC.MODE`, `CAP.PREC.ALT`, `CAP.PREC.STAT`, and `CAP.PREC.EXC.{EN,ST}` as latched at decode.
* Effective PET/EW/ACCW, rounding, NaN policy, quantization, SAE defaults, and ZMODE apply uniformly across all RSV lanes; RSV defines no local variants.
* RSV lane-level exceptions update CAP sticky status bits in the same manner as scalar execution.

### 9.2 Masked Writeback Policy (ZMODE)

| ZMODE | Behavior                                                |
| ----- | ------------------------------------------------------- |
| 0 (merge)   | masked-off lanes keep their old `rd` element            |
| 1 (zeroing) | masked-off lanes write logical zero according to EFF_EW |

* Masked writeback for all RSV-affected instructions (including optional profiles) follows `CAP.PREC.MODE.ZMODE`.
* Predication suppresses writeback for masked lanes; ZMODE determines whether suppressed lanes merge or zero.

### 9.3 Floating-Point Exceptions & Sticky Flags

* Only the first faulting lane produces an architecturally visible trap, subject to `CAP.PREC.EXC.EN` and effective SAE (`EFF_SAE`).
* SAE asserted via `svon.fpctl` or `CAP.PREC.MODE.SAE_DEF` suppresses traps but must still update sticky bits in `CAP.PREC.EXC.ST` and status in `CAP.PREC.STAT`.
* `SVFAULTI` records the index of the first faulting lane.

### 9.4 Memory and Domain Defaults (XMEM Integration)

* All RSV memory-touching operations reuse XMEM defaults for coherence (`CAP.XMEM.DOM`), class mapping, streaming profiles, compression, and descriptor policies.
* Gather/scatter uses SVMEM configuration (`SVMBASE/SVMIDX/SVMSCL/SVMLEN/SVMMODE`). Masked-off lanes do not generate memory requests.
* If `SVMLEN=0`, element width defaults to `CAP.PREC.STAT.EFF_EW`.

### 9.5 Mandatory Rules for RSV Gather/Scatter

1. Predication: masked-off lanes must not generate memory requests.
2. ZMODE: masked writeback follows CAP merge/zeroing semantics.
3. FOF: `SVMSCL.FOF=1` behaves identically to other domains (RT, MTX, GFX).
4. Domain: coherence domain defaults to `CAP.XMEM.DOM` unless overridden via descriptor.

### 9.6 Descriptor Interoperability (`XPHMG_MEMCTL`)

* When RSV operations are triggered via descriptors (e.g., in GFX or NPU pipelines), only explicitly set fields (`dom`, `mclass`, `sprof_id`, `hyb`, `sec`) override CAP defaults.
* Unspecified fields fall back to `CAP.XMEM.DESCPOL` and CAP defaults; RSV must not diverge silently from system memory policy.

### 9.7 Optional Profiles (PRMT, CEXP, S-SAT)

* Optional RSV profiles inherit CAP precision rules: effective PET/EW/ACCW, saturation behavior (`SAT_HIT`), and quantization flags (`EFF_Q`).
* Compaction/expansion obey ZMODE and use XMEM SVMEM for gather/scatter forms; masked elements follow merge/zeroing semantics.
* Permutation and widening/narrowing operations must preserve FP semantics and record `DOWNCAST_TAKEN` or `UNSUP_FMT` in CAP status when narrowing outside native support.

### 9.8 State-Change Latching

* CAP precision changes (`APPLY0/1` in MODE/ALT) update effective state atomically; RSV must latch CAP state at instruction boundaries and must not observe partial updates.
* Per-instruction overrides via `svon.fpctl` apply only to the next instruction and do not modify CAP state.

### 9.9 Notes for Implementers (Informative)

* RSV engines should latch CAP/XMEM state at decode and retain it for the duration of the RSV-affected instruction, even if system software modifies CAP/XMEM concurrently.
* Debug/replay tools should expose both RSV state and the latched CAP/XMEM state to enable deterministic re-execution (see Section 10 for cross-domain debug rules).

---

## 10  Cross-Domain Interoperability (Normative)

### 10.1 General Principles

* Cross-domain consistency is enforced by shared CAP/XMEM semantics: RSV must not redefine precision, exceptions, or memory behavior when interoperating with MTX, RT, GFX, ACCEL, or scalar domains.
* Data handed off between domains must not require reinterpretation when CAP/XMEM state is respected and layout metadata is preserved.

### 10.2 Consumption of XDL Tiles Produced by MTX

* RSV engines must be capable of consuming Tier-0 regions written by MTX in canonical XDL tile layouts (`XDL_TILE4x4_ROWMAJOR` or composite tile) when `tier=T0`, `mclass=CLS_SHARED`, `dom=DOM_RSV`.
* Permitted RSV interpretations:
  * Row vector view: each row is a 4-element vector.
  * Flattened view: the tile is treated as a 16-element RSV vector.
  * Column vector view: optional, based on SVMEM stride rules.
* RSV must not reorder or reinterpret tile contents beyond these permitted views.

### 10.3 RSV <-> MTX

* RSV can prepare MTX tiles (layout, type-packing) without precision mismatches because numeric policy is governed by CAP.
* MTX outputs can be consumed by RSV without reinterpretation when layout metadata indicates compatibility; ZMODE and precision behavior follow CAP.

### 10.4 RSV <-> RT

* Per-lane geometry operations and visibility masks computed in RSV inherit CAP/XMEM semantics; RT FOF, ZMODE, and SVMEM remain consistent with RSV behavior.
* No RSV-local exception or precision model is introduced; RT observes CAP-derived behavior.

### 10.5 RSV <-> GFX

* RSV permutations may reorganize vertex/pixel tensors for texture or shading ops; GFX texture gather must behave identically to RSV gather with the same CAP.XMEM state.
* Streaming and coherence defaults remain governed by XMEM; RSV must not introduce domain-specific memory semantics.
* Mesh-style primitive emission is a software-kernel pattern: RSV provides control flow and workgroup orchestration while GFX provides visibility/compaction helpers; no GFX-owned mesh execution model is defined.

### 10.6 RSV <-> Scalar ALU

* Scalar fallback must produce results bit-equivalent to RSV lane 0 for the same operands and CAP/XMEM state.
* Disabling RSV restores scalar execution; no lingering RSV state may alter scalar outcomes.

### 10.7 Debug / Replay and State Exposure

* Implementations must expose RSV state (SVSTATE incl. PBANK, SVDST, SVMEM) and the latched CAP.PREC/ST/XMEM state to debug and replay tools; PMASK state is exposed via XPHMG_PMASK.
* Exposure must align with instruction boundaries to permit deterministic replay when CAP precision or rounding modes change dynamically.

### 10.8 Privilege and Delegation

* All RSV CSRs are Machine-mode read/write; delegation to lower privilege is implementation-defined but must preserve architectural visibility of SVSTATE and SVFAULTI.
* Debuggers and trace units must provide access to SVSTATE and the latched CAP state per instruction boundary.

---

## 11  Optional Profiles and Extensions (Informative/Normative as tagged)

### 11.1 Overview

Implementations may advertise support for any subset of the optional RSV profiles listed below. Optional profiles do not change the core RSV programming model; they add instruction families that consume existing SV CSRs and CAP/XMEM state.

### 11.2 Profile Groups

* **XRSV-PRMT** (*Permute & Table*): `svperm.v`, `svpermi2.v`, `svshuffle.v`, `svzip.{lo,hi}`, `svunzip.{lo,hi}`, `svext.v`, `svtbl.v`, `svtbx.v`.
* **XRSV-CEXP** (*Compact/Expand*): `svcompress`, `svexpand`, `svscat.cmp`, `svgath.exp`, `svpopcnt.pm`.
* **XRSVS** (*Saturating & Widen/Narrow*): `svadd.sat.{s,u}`, `svsub.sat.{s,u}`, `svabs.sat.s`, `svmul.wide.{s,u}`, `svmla.wide.{s,u}`, `svnarrow.sat.{s,u}`, `svavg.rnd.{s,u}`, `svmin.{s,u}`, `svmax.{s,u}`, `svshift.wide.s`.

| Extension | Description                                                       |
| --------- | ----------------------------------------------------------------- |
| `XRSVR`   | Adds Reduction operations.                                        |
| `XRSVS`   | Adds Saturation and Width override.                               |
| `XRSVGZ`  | Allows writable x0 during SV execution.                           |

### 11.3 Architectural Invariants

* All optional-profile instructions obey RSV predication (PMASK, per `pbank_eff`) and masked-lane writeback policy `CAP.PREC.MODE.ZMODE`.
* Integer saturation must set `CAP.PREC.STAT.SAT_HIT`; FP behavior follows CAP precision rules.
* Memory-touching forms consume `CAP.XMEM.DOM` and may use streaming profiles from `CAP.XMEM.SPROF*`; masked lanes must not generate memory requests.
* Unsupported element widths or operations must raise *Illegal Instruction*.

### 11.4 Interaction with CAP/XMEM

* Precision, rounding, saturation, NaN, and quantization behavior are inherited from CAP; no profile-specific numeric policy is defined.
* Gather/scatter forms in PRMT/CEXP use XMEM SVMEM configuration; ZMODE governs masked writeback.
* Descriptor-triggered usage follows the descriptor override rules in Section 9; unspecified fields fall back to CAP defaults.

### 11.5 Capability Discovery and Enumeration

* Advertisement of supported profiles is implementation-defined and should follow the project's capability discovery mechanism (e.g., `XPHMG_RSV-OP`/custom-1 encoding presence).
* Absence of a profile implies all associated instructions must trap as *Illegal Instruction*.

### 11.6 Notes for Implementers (Informative)

* Optional profiles share the same SV CSR state; no additional RSV CSRs are defined by these profiles in v0.1.
* Widening/narrowing and permutation operations that exceed native format support must report status in CAP (`DOWNCAST_TAKEN`, `UNSUP_FMT`) as described in Section 9.

---

## 12  Encoding Summary (Informative)

### 12.1 Base RSV Prefix Encodings

| Opcode    | Meaning                 | Type              |
| --------- | ----------------------- | ----------------- |
| `0001011` | RSV prefix instructions | I-Type (custom-0) |
| `1110011` | CSR access (SYSTEM)     | I-Type            |

* Prefix opcodes (`custom-0`) cover `svsetvl`, `svon.one`, `svon.blk`, `svend`, `svp.one.vlstep`, and `svon.fpctl` as defined in Section 6.
* Prefixes operate in the scalar pipeline and configure SVSTATE and related CSRs for subsequent RSV-affected instructions.

### 12.2 Optional RSV Profile Encodings (custom-1)

* Optional RSV profile instructions (`XRSV-OP`) are encoded under **custom-1 (`0101011`)** and grouped by `funct3` as defined in `XPHMG_RSV-OP`.
* Widening ops use the **pair-dest convention**: when the instruction's `PAIR` bit is set, results are written to `rd` (low) and `rd^1` (high); this is legal only if `rd` is even and `SVDST.stride==1`.
* Absence of a supported profile implies all corresponding encodings must trap as *Illegal Instruction*.

### 12.3 Reserved and Undefined Encodings

* Encodings not defined in this specification under `custom-0` or `custom-1` are reserved and must trap as *Illegal Instruction*.
* Reserved step codes (`bbb`/`ddd` = `1xx`) in `svp.one.vlstep` are reserved and must not introduce additional semantics in v0.1.

### 12.4 Notes for Implementers (Informative)

* Implementations should align decode logic with the `custom-0`/`custom-1` split to avoid aliasing with standard ISA or V-extension encodings.
* Capability discovery for optional profiles should follow the project's agreed mechanism (e.g., checking presence of `XPHMG_RSV-OP` support) before emitting custom-1 encodings.

---

## 13  Programming Examples (Informative)

### 13.1 Vector Add 3D (Integer)

Illustrates a minimal one-shot RSV sequence over three lanes with default CAP/XMEM state.

```asm
# r10..r12 = A(x,y,z)
# r20..r22 = B(x,y,z)
# r30..r32 = Result
svsetvl x0, 3           # VL=3
svon.one                # enable for next instruction
add x30, x10, x20       # lane-wise add over r10..12 and r20..22 into r30..32
```

### 13.2 Scalar -> Vector (Broadcast, FP)

Demonstrates broadcast of a scalar operand with FP control override (SAE + zeroing) for one instruction.

```asm
svsetvl x0, 16          # VL=16
csrw  SVSRCA, x10       # base A (zmm2)
csrw  SVSRCB, x20       # base B (zmm3)
csrw  SVDST, x30        # base D (zmm1)
svon.fpctl rc=RNE, sae=1, z=1   # one-shot FP override
svon.one
fadd.s x30, x10, x20    # broadcast x20, zero masked-off lanes per ZMODE
```

### 13.3 Predicate-Controlled Masked Operation (Zero/Merge via CAP)

Shows predicated FP addition with mask-controlled writeback following CAP.ZMODE.

```asm
# VL = 16, predication via PMASK (pbank_eff), zeroing active
svsetvl x0, 16
csrrw  x0, SVSRCA, x10  # base A (zmm2)
csrrw  x0, SVSRCB, x20  # base B (zmm3)
csrrw  x0, SVDST, x30   # base D (zmm1)
svon.fpctl rc=RNE, sae=1, z=1   # one-shot override
svon.one
fadd.s x30, x10, x20    # executes 16 lanes, masked-off lanes zeroed
```

### 13.4 Scalar -> Vector with Suppressed FP Exceptions

Illustrates SAE in effect for one FP multiply; traps are suppressed but sticky bits update per CAP.

```asm
# Multiply scalar r5 by vector r10..r13, suppress FP exceptions
svsetvl x0, 4
csrrw x0, SVSRCA, x10
csrrw x0, SVSRCB, x5
csrrw x0, SVDST, x10
svon.fpctl rc=RTZ, sae=1, z=0
svon.one
fmul.s x10, x10, x5
```

NOTE: All examples assume CAP/XMEM defaults are configured appropriately (e.g., EFF_EW matches operand width, DOM/CLS settings permit the accesses) and that optional CSRs used (e.g., SVSAT) are implemented; otherwise, behavior falls back to the defaults defined in Sections 5-9.

---

## 14  Implementation Notes (Informative)

* Internal micro-loop execution can be pipelined or sequenced; the architectural requirement is to preserve decode-time snapshots of SV and CAP/XMEM state for each RSV-affected instruction.
* Stride counters and base register arithmetic may wrap modulo the scalar register file width (XLEN/ABI-defined register count); overlapping windows programmed by software are architecturally permitted.
* Minimal hardware may choose small VL limits (e.g., VL >= 8) and restricted stride sets (e.g., stride in {0,1}); all such limits must still honor the architectural rules for clamping and reporting effective VL via `svsetvl`.
* Optional features (Predication, Reduction, Saturation) are controlled by SV CSRs and PMASK; unimplemented optional SV CSRs must behave as specified (read-as-zero/ignore for saturation override).
* Implementations may cache or shadow SV CSR fields internally for timing; architectural visibility on reads/writes and decode-time sampling must remain consistent with Sections 5-7.
* Illegal or reserved encodings must trap rather than degrade silently to ensure compliance and debuggability.

NOTE: Microarchitectural choices such as unrolling, lane grouping, or fused execution are implementation-defined; they must not change architecturally visible ordering, CAP/XMEM state latching, or fault/exception semantics defined in Sections 7-9.

---

## 15  Acknowledgments

This extension draws inspiration from the *Simple-V (SVP64)* work by **Luke Kenneth Casson Leighton** and the **Libre-SOC** project. The design philosophy of *vectorization by reinterpretation* was preserved and adapted to fit the RISC-V encoding space and the broader XPHMG architecture (CAP, XMEM, MTX, RT, GFX, ACCEL).

---

## 16  References

* Libre-SOC Simple-V / SVP64 Specification - [https://libre-soc.org/openpower/sv/svp64/](https://libre-soc.org/openpower/sv/svp64/)
* RISC-V Unprivileged ISA Volume I (2024)
* RISC-V Vector Extension (V1.0)
* XPHMG_CAP - Precision, Exception, and Memory Control Specification
* XPHMG_XMEM - Unified LDS / Cache / Streaming Memory Control

---

Designed for integration with `XPHMG_CAP`, and `XPHMG_XMEM`.

*Licensed under CC-BY-SA 4.0 - Open Specification maintained by Pedro H. M. Garcia.*
