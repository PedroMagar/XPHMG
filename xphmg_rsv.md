# RISC-V Scalar-Vectors (XPHMG_RSV)

**Category:** Vendor Extension (`XPHMG_RSV`)
**Version:** 0.1.0
**Author:** Pedro H. M. Garcia
**Based on concepts by:** Luke Kenneth Casson Leighton (*Libre-SOC / Simple-V*)
**License:** CC-BY-SA 4.0 (open specification)

> This document describes an experimental vendor extension of the RISC-V ISA that adapts the *Simple-V* (SVP64) model created by Luke Kenneth Casson Leighton for Libre-SOC.
> The purpose is to enable lightweight vectorization by reinterpretation of existing scalar instructions while remaining binary-compatible with standard RISC-V cores when disabled.

---

## 1  Overview (Informative)

`XPHMG_RSV` (RISC-V **Scalar-Vectors**) defines a lightweight **scalar-vector reinterpretation layer** for the RISC-V ISA.
Rather than introducing a separate vector register file or a distinct execution model, RSV allows existing **scalar instructions** to be executed repeatedly over a programmable window of registers, under explicit control of vector length, stride, and predicate masks.

RSV is designed as a *structural adaptation* of scalar code, not as a replacement for the standard RISC-V Vector (V) extension. When disabled, all instructions execute with their original scalar semantics and remain fully compatible with unextended RISC-V cores.

Unlike classical vector ISAs, RSV does not define its own numeric or memory policy. All precision, rounding, saturation, exception handling, masking semantics, and gather/scatter behavior are **architecturally inherited** from `XPHMG_CAP`. At each instruction boundary, an RSV engine must snapshot the effective `CAP.PREC.*` and `CAP.XMEM.*` state, ensuring consistent behavior across RSV, MTX, RT, GFX, and ACCEL domains.

RSV is intended to provide:

* Compact vectorization for graphics, geometry, and compute workloads,
* Efficient per-lane control using predicate masks and zero/merge semantics,
* Seamless interoperability with XPHMG tile (`MTX`) and memory (`XMEM`) models,
* A predictable, low-complexity implementation path for heterogeneous accelerators.

This extension is inspired by the *Simple-V* concept developed in the Libre-SOC project, but is specified independently and integrated tightly into the XPHMG architectural model.

---

## 2  Compatibility

* Works with **RV32I/RV64I** and all standard extensions.
* Coexists with the RISC-V V extension.
* When disabled, instructions behave exactly as defined by the base ISA.
* CSRs occupy the vendor range **0x7F8 – 0x7FF**.

1. “RSV gather/scatter must follow SVMEM from XMEM. It may not define its own gather logic.”
2. “RSV masked writeback uses `CAP.PREC.MODE.ZMODE`; this affects MTX and RT when using RSV to prepare data.”
3. “FP exceptions follow CAP global semantics; RSV does not invent its own exception model.”

### 2.1 Precision and Exception Control

RSV interprets scalar FP and integer instructions according to `CAP.PREC.MODE` and `CAP.PREC.STAT`:
> *Field names reflect CAP; if an implementation uses different names, this specification treats them as aliases.*

### 2.2 Memory and Domain Defaults

All RSV gather/scatter and stream-like operations must respect:

* `CAP.XMEM.DOM` — default coherence domain;
* `CAP.XMEM.SVM*` — base/scale/index configuration for indexed addressing;
* `CAP.XMEM.CAP.*` bits — feature presence (e.g. compression, security, hybrid mode).

---

## 3  Programmer’s Model

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
| **pred[i]**  | Predicate mask bit (optional)        |

`inc=0` → broadcast. VL ≥ 1.

---

## 4  CSRs (0x7C0 – 0x7C7)

| Addr  | Name         | Purpose                                                             |
| ----- | ------------ | ------------------------------------------------------------------- |
| 0x7F8 | **SVSTATE**  | Enable bit `EN`, ONE_SHOT, BLK, VL, mode flags (pred/reduce/width). |
| 0x7F9 | **SVSRCA**   | Source A base + stride.                                             |
| 0x7FA | **SVSRCB**   | Source B base + stride.                                             |
| 0x7FB | **SVDST**    | Destination base + stride.                                          |
| 0x7FC | **SVPMASK0** | Predicate bits [31:0].                                              |
| 0x7FD | **SVPMASK1** | Predicate bits [63:32].                                             |
| 0x7FE | **SVSAT**    | Saturation / element-width override.                                |
| 0x7FF | **SVFAULTI** | Index of element that faulted.                                      |

Minimal implementations may omit masks and saturation:

* If `SVPMASK*` are not implemented, they must read as all-ones and ignore writes.
* If `SVSAT` is not implemented, it reads zero and is ignored; saturation must still be reported via `CAP.PREC.STAT.SAT_HIT` where applicable.

---

## 5  Prefix Instructions (custom-0 opcode = `0001011`)

| imm[11:0]      | rs1     | funct3 | rd      | opcode    | Mnemonic             | Description                                                                       |
| -------------- | ------- | ------ | ------- | --------- | -------------------- | --------------------------------------------------------------------------------- |
| `000000000000` | `xxxxx` | 000    | `xxxxx` | `0001011` | **svsetvl rd, rs1**  | Set VL = `rs1[7:0]`; return effective VL in `rd`.                                 |
| `0000xxxxxxxx` | `00000` | 000    | `xxxxx` | `0001011` | **svsetvl rd, imm8** | Immediate VL (1–256).                                                             |
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

**Encoding semantics:**

zzzzz: `imm[4:0] = rc[2:0], sae, z`
```
imm[4:2] → RC (0=RNE,1=RTZ,2=RDN,3=RUP,4=RMM)
imm[1]   → SAE (1 = suppress exceptions)
imm[0]   → ZMODE (1 = zeroing, 0 = merge)
```

**Operational semantics:**

* Applies only to the immediately following instruction (similar to `svon.one`).
* If omitted, the instruction uses `CAP.PREC.MODE.{FP_RMODE,SAE_DEF,ZMODE}`.
* On the following instruction, defaults revert automatically.
* A conformant implementation may ignore unimplemented fields (e.g. if only FP32 supported).

### 5.1 Consumption of XDL Tiles Produced by MTX

RSV engines MUST be capable of consuming Tier-0 regions written by MTX in the canonical XDL tile layout.

A region qualifies for MTX→RSV consumption if:

- `tier = T0`
- `mclass = CLS_SHARED`
- `dom = DOM_RSV` after handoff
- XDL layout metadata indicates a tile layout (`XDL_TILE4x4_ROWMAJOR` or a composite tile)

RSV MAY interpret the tile in multiple vector forms:

- **Row vector view** — each row is a 4-element vector.  
- **Flattened view** — the tile is treated as a 16-element RSV vector.  
- **Column vector view** — optional, based on `SVMEM` stride rules.

RSV MUST NOT reorder or reinterpret tile contents beyond these permitted views.

---

## 6 Execution Semantics

* `svon.one` sets `SVSTATE.EN=1, ONE_SHOT=1`.
* `svon.blk N` sets `EN=1, ONE_SHOT=0, BLK=N`.
* Each instruction executed while `EN=1` uses a snapshot of all SV CSRs at decode.
* After each instruction, `BLK` decrements; when `BLK==0`, `EN` clears.
* `svend` forces `EN=0`.

---

## 7  Examples

### 7.1 Vector Add 3D

```asm
# r10..r12 = A(x,y,z)
# r20..r22 = B(x,y,z)
# r30..r32 = Result
svsetvl x0, 3
svon.one
add x30, x10, x20
```

### 7.2 Scalar × Vector (Broadcast)

```asm
svsetvl x0, 16
csrw  0x7C1, x10    # base A (zmm2)
csrw  0x7C2, x20    # base B (zmm3)
csrw  0x7C3, x30    # base D (zmm1)
svon.fpctl rc=RNE, sae=1, z=1
svon.one
fadd.s x30, x10, x20
```

### 7.3 Example snippet — predicate-controlled masked operation (zero/merge semantics) using RSV + CAP

```asm
# Example: vector add with per-lane predicate mask and zero-fill on masked-off lanes
# VL = 16, mask in SVPMASK0, zeroing active

svsetvl x0, 16
csrrw  x0, 0x7C1, x10    # base A (zmm2)
csrrw  x0, 0x7C2, x20    # base B (zmm3)
csrrw  x0, 0x7C3, x30    # base D (zmm1)
svon.fpctl rc=RNE, sae=1, z=1   # one-shot override
svon.one
fadd.s x30, x10, x20             # executes 16 lanes, zeroed where mask=0
```


### 7.4 Example — scalar × vector with suppressed exceptions

```asm
# Multiply scalar r5 by vector r10..r13, suppress FP exceptions
svsetvl x0, 4
csrrw x0, 0x7C1, x10
csrrw x0, 0x7C2, x5
csrrw x0, 0x7C3, x10
svon.fpctl rc=RTZ, sae=1, z=0
svon.one
fmul.s x10, x10, x5
```

---

## 8  Exception Handling

On the first faulting element *i*, the hardware stores *i* in `SVFAULTI` and signals a trap as if a scalar instruction failed once.
For floating-point operations, exception signaling obeys the following rules:

1. If `svon.fpctl.sae=1` or `CAP.PREC.MODE.SAE_DEF=1`, suppress all synchronous FP traps; Sticky bits in `CAP.PREC.EXC.ST` must still be set, and may be read after the vectorized instruction completes.
2. Otherwise, if `CAP.PREC.EXC.EN` enables the raised condition (`NV,DZ,OF,UF,NX`), deliver the corresponding trap.
3. Software can poll sticky bits in `CAP.PREC.EXC.ST` for asynchronous reporting.
4. Implementations must update `CAP.PREC.STAT.EFF_SAE` to reflect the effective mode active during the fault.
5. Implementations should not raise multiple traps for multiple lanes; they must record the first faulting element in `SVFAULTI`.

---

## 9  Privilege and Debug

* All RSV CSRs are Machine-mode RW.
* Implementations may delegate to lower privilege.
* Debug tools must expose SVSTATE and SVFAULTI.
* The CAP snapshot reflects `CAP.PREC.STAT` **after** bank selection and `svon.fpctl` overrides have been applied.

### 9.1 Cross-domain synchronization
Debuggers and trace units must expose both `SVSTATE` and the current latched `CAP.PREC.STAT` snapshot per instruction boundary.
This ensures consistent replay when CAP precision or rounding modes change dynamically.

---

## 10  Encoding Summary

| Opcode    | Meaning                 | Type              |
| --------- | ----------------------- | ----------------- |
| `0001011` | RSV prefix instructions | I-Type (custom-0) |
| `1110011` | CSR access (SYSTEM)     | I-Type            |

**Optional RSV Profiles (XRSV-OP):**
All XRSV-OP instructions are encoded under **custom-1 (`0101011`)** and grouped by `funct3` as defined in “XPHMG_RSV-OP”. The base RSV prefixes (`svsetvl`, `svon.*`, `svend`) remain under **custom-0 (`0001011`)**. Widening ops use the **pair-dest convention**: when the instruction’s `PAIR` bit is set, results are written to `rd` (low) and `rd^1` (high); this is legal only if `rd` is even and `SVDST.stride==1`.

---

## 11 Optional Profiles (non-mandatory)

Implementations **may** advertise support for any subset of:

* **XRSV-PRMT** (*Permute & Table*): `svperm.v`, `svpermi2.v`, `svshuffle.v`, `svzip.{lo,hi}`, `svunzip.{lo,hi}`, `svext.v`, `svtbl.v`, `svtbx.v`.
* **XRSV-CEXP** (*Compact/Expand*): `svcompress`, `svexpand`, `svscat.cmp`, `svgath.exp`, `svpopcnt.pm`.
* **XRSVS** (*Saturating & Widen/Narrow*): `svadd.sat.{s,u}`, `svsub.sat.{s,u}`, `svabs.sat.s`, `svmul.wide.{s,u}`, `svmla.wide.{s,u}`, `svnarrow.sat.{s,u}`, `svavg.rnd.{s,u}`, `svmin.{s,u}`, `svmax.{s,u}`, `svshift.wide.s`.

All instructions obey RSV predication (`SVPMASK*`) and masked-lane writeback policy `CAP.PREC.MODE.ZMODE`. Integer saturation sets `CAP.PREC.STAT.SAT_HIT`. Memory-touching forms consume `CAP.XMEM.DOM` and may use streaming profiles from `CAP.XMEM.SPROF*`. Unsupported element widths/operations must raise *Illegal Instruction*.

---

## 12  Implementation Notes

* Internal micro-loop can be pipelined.
* Stride counters may wrap modulo 32 or 64 registers.
* Minimal hardware: `VL≤8`, `stride∈{0,1}`.
* Optional features (Predication, Reduction, Saturation) controlled by CSRs.

---

## 13  Optional Predication Enhancements

| Extension | Description                                                       |
| --------- | ----------------------------------------------------------------- |
| `XRSVP`   | Adds Predication and optional masked zero/merge (uses CAP.ZMODE). |
| `XRSVR`   | Adds Reduction operations.                                        |
| `XRSVS`   | Adds Saturation and Width override.                               |
| `XRSVGZ`  | Allows writable x0 during SV execution.                           |

When `XRSVP` is present, predicate masks map to `SVPMASK*`; mask-off behavior follows `ZMODE` (merge/zero) per §6.

---

## **15. Cross-Domain Interoperability & Implementation Requirements (Normative)**

This section extends RSV with the same interoperability guarantees already defined for CAP and XMEM, ensuring that **RSV, MTX, RT, GFX, scalar ALU, and NPU domains behave consistently** under the unified XPHMG precision and memory model.

This section is **normative** unless otherwise specified.

### **15.1 Precision / Numeric State (CAP → RSV)**

Every RSV-enabled instruction (including prefix-activated expansions) **must snapshot the full effective precision state** at decode:

* `CAP.PREC.MODE`
* `CAP.PREC.ALT`
* `CAP.PREC.STAT`
* `CAP.PREC.EXC.{EN,ST}`

The effective PET/EW/ACCW, saturation rules, rounding mode, FTZ, Q/UNS, NaN policy, SAE defaults, packed-INT rules and ZMODE **must be applied uniformly across all RSV lanes**.

RV implementations **must not** introduce RSV-local variants of:

* rounding behavior,
* NaN canonicalization,
* saturating conversions,
* exception rules,
* mixed FP8/BF16/FP16 accumulation mappings,
* quantization with ZP/SCALE.

All of this is delegated to CAP, and RSV only consumes the **effective state**.

### **15.2 ZMODE (Merge vs Zeroing) for Masked Lanes**

RSV masked writeback **must obey CAP.PREC.MODE.ZMODE**:

| ZMODE           | Behavior                                                |
|  | - |
| **0 – merge**   | masked-off lanes keep their old `rd` element            |
| **1 – zeroing** | masked-off lanes write logical zero according to EFF_EW |

This rule applies to:

* scalarized ALU ops executed under RSV,
* FP operations,
* RSV-profiles (permutation, saturating, compact/expand),
* RSV gather/scatter (when using SVMEM).

No RSV instruction is allowed to define its own masked-lane semantics.

### **15.3 FP Exceptions & Sticky Flags**

RSV lane-level exceptions must update:

* `CAP.PREC.STAT.{SAT_HIT,FTZ_HIT,DOWNCAST_TAKEN,UNSUP_FMT}`
* `CAP.PREC.EXC.ST.{NV,DZ,OF,UF,NX,QNAN_SEEN,SNAN_SEEN}`

Rules:

1. **Only the first faulting lane produces an architecturally visible trap**, iff:

   * the exception type is enabled in `EXC.EN`,
   * **and** effective SAE (`EFF_SAE`) = 0.

2. When SAE override (via `svon.fpctl`) = 1:

   * the trap is suppressed,
   * sticky bits are still updated.

3. `SVFAULTI` must store the lane index of the first failing element.

4. A domain cannot change its FP exception model. All must follow CAP semantics.

### **15.4 Uniform Memory Behavior via XMEM**

RSV memory instructions (including RSV-expanded loads/stores, RSS profiles with gather/scatter, and XMEM-touching ops issued inside vector loops) **must obey the following architectural defaults** from CAP.XMEM:

* coherence domain (`CAP.XMEM.DOM`),
* memory class mapping (`CAP.XMEM.CLSMAP`),
* streaming detector and profiles (`STREAM_CTL`, `SPROF0..3`),
* inline compression policy (`COMP_CTL`),
* descriptor defaults (`DESCPOL`),
* gather/scatter parameters (SVMBASE/SVMIDX/SVMSCL/SVMLEN/SVMMODE).

#### **Mandatory rules for RSV gather/scatter**

1. **Predication**: masked-off lanes must not generate memory requests.
2. **ZMODE**: masked writeback must follow merge/zeroing semantics from CAP.
3. **FOF**: `SVMSCL.FOF=1` must behave exactly like any other domain (RT, MTX, GFX).
4. **Element width**: if `SVMLEN=0`, use `CAP.PREC.STAT.EFF_EW`.
5. **Domain**: default coherence follows `CAP.XMEM.DOM` unless overridden via descriptor.

RSV **cannot** implement a gather/scatter path that diverges from XMEM SVMEM rules.

### **15.5 Descriptor Interoperability (`XPHMG_MEMCTL`)**

If RSV operations are triggered via a descriptor (e.g., in a GFX or NPU pipeline):

* Only fields explicitly set (`dom`, `mclass`, `sprof_id`, `hyb`, `sec`) override CAP.
* All unspecified fields fall back to `CAP.XMEM.DESCPOL` and CAP defaults.

This ensures RSV never silently diverges from the system’s global memory policy.

### **15.6 Permute, Compact, Saturation Profiles (RSV-Profiles)**

All RSV optional profiles (PRMT, CEXP, S-SAT) are extended by the following rules:

#### **15.6.1 Precision & saturating arithmetic**

All saturating arithmetic in RSV-Profiles must:

* use effective PET/EW/ACCW,
* set `CAP.PREC.STAT.SAT_HIT` when saturation occurs,
* honor quantization rules if `EFF_Q = 1`,
* obey ZMODE for masked lanes.

#### **15.6.2 Compaction / expansion (CEXP)**

RSV compaction must adhere to CAP:

* masked elements = merge/zeroing via ZMODE,
* gather/scatter variants must use SVMEM from CAP.XMEM,
* FOF must behave consistently with other domains.

#### **15.6.3 Permutation profiles (PRMT)**

Any lane-shuffling, widening/narrowing or AOS↔SOA reshaping must:

* preserve FP semantics exactly,
* follow ZMODE rules,
* record `DOWNCAST_TAKEN` or `UNSUP_FMT` in STAT if narrowing is outside native support.

### **15.7 Cross-Domain Dataflow Guarantees**

Because RSV now follows the unified CAP+XMEM model, the following guarantees hold:

#### **15.7.1 RSV ↔ MTX**

* RSV can prepare MTX tiles (layout, type-packing) without precision mismatches.
* MTX outputs can be consumed by RSV without reinterpretation.

#### **15.7.2 RSV ↔ RT**

* BVH visibility masks and per-lane geometry ops computed in RSV inherit proper FP/INT semantics.
* RT-side FOF, ZMODE, and SVMEM remain consistent with RSV behavior.

#### **15.7.3 RSV ↔ GFX**

* RSV permutations can reorganize vertex/pixel tensors for texture or shading ops.
* GFX texture gather must behave identically to RSV gather with the same CAP.XMEM state.

#### **15.7.4 RSV ↔ Scalar ALU**

* Scalar fallback must produce results bit-equivalent to RSV lane 0.

### **15.8 State-Change Rules (Apply semantics)**

RSV is not allowed to partially observe CAP writes.

1. CAP precision changes (`APPLY0/1` in MODE/ALT) update the effective state **atomically**.
2. RSV engines must latch CAP state at instruction boundary.
3. Per-instruction overrides (`svon.fpctl`) must not modify global CAP state.
4. Override is valid **only for the next instruction**, then reverts.

### **15.9 Debug / Replay Rules**

Implementations must:

* expose RSV state (SVSTATE, masks, SVDST, SVMEM) and the **latched** CAP.PREC/ST/XMEM state,
* ensure replay tools see stable precision/memory configurations per boundary,
* apply SAE/ZMODE consistently for all lanes.

---

## 16 Acknowledgments

This extension is inspired by the *Simple-V (SVP64)* work from
**Luke Kenneth Casson Leighton** in the **Libre-SOC** project.
The design philosophy of *vectorization by reinterpretation* is preserved
and adapted for the RISC-V encoding space.

---

## 17 References

* Libre-SOC Simple-V / SVP64 Specification — [https://libre-soc.org/openpower/sv/svp64/](https://libre-soc.org/openpower/sv/svp64/)
* RISC-V Unprivileged ISA Volume I (2024)
* RISC-V Vector Extension (V1.0)
* XPHMG_CAP - Precision, Exception, and Memory Control Specification
* XPHMG_XMEM - Unified LDS / Cache / Streaming Memory Control

---

Designed for integration with `XPHMG_CAP`, and `XPHMG_XMEM`.

*Licensed under CC-BY-SA 4.0 — Open Specification maintained by Pedro H. M. Garcia.*
