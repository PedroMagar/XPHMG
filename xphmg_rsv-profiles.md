# **XPHMG_RSV-Profiles**

## Optional Scalar-Vector Profiles for XPHMG_RSV

**Category:** Vendor Extension Addendum (`XPHMG_RSV-Profiles`)
**Version:** 0.1.0
**Author:** Pedro H. M. Garcia
**License:** CC-BY-SA 4.0 (open specification)

---

## **1. Overview (Informative)**

`XPHMG_RSV-Profiles` defines a set of **optional instruction profiles** that extend the base `XPHMG_RSV` (Scalar-Vectors) execution model.

These profiles introduce commonly needed scalar-vector operations — such as permutation, compaction/expansion, and saturating or widening arithmetic — without altering the architectural contract of RSV or the authority of `XPHMG_CAP` and `XPHMG_XMEM`.

Support for RSV profiles is **entirely optional**.
An implementation may support none, some, or all profiles independently.
If a profile-specific instruction is not supported, it must trap as an *illegal instruction*.

---

### **1.1 Relationship to XPHMG_RSV**

The base `XPHMG_RSV` extension defines:

* scalar-vector reinterpretation,
* vector length and register windows,
* predicate masks and masking semantics,
* interaction with CAP-controlled numeric policy and XMEM-controlled memory behavior.

`XPHMG_RSV-Profiles` **does not modify** any of these rules.

All profiles:

* reuse the existing RSV enablement (`svon.*`),
* operate on the same register windows and masks (`SVSTATE`, `SVPMASK*`, `SVSRC*`, `SVDST`),
* inherit numeric precision, rounding, saturation, quantization, exception handling, and ZMODE behavior from `XPHMG_CAP`,
* inherit memory behavior, domains, and gather/scatter semantics from `XPHMG_XMEM`.

No profile introduces new global CSRs, new numeric modes, or profile-specific memory rules.

---

### **1.2 Purpose of Profiles**

The profiles defined in this document serve three main purposes:

1. **Encapsulation of common idioms**
   Frequently used scalar-vector patterns (e.g., permutation, compaction, widening arithmetic) are grouped into named profiles rather than being folded into the RSV base.

2. **Incremental implementation**
   Implementers may choose to support only the RSV base, or selectively add profiles according to target workload, silicon budget, or performance goals.

3. **Architectural clarity**
   By isolating higher-level operations into optional profiles, the RSV base remains compact, predictable, and easy to reason about.

Profiles are therefore **extensions of convenience**, not architectural requirements.

---

### **1.3 Normative Scope**

Unless explicitly stated otherwise:

* All instructions defined in this document are **optional**.
* Lack of support must result in an *illegal instruction* exception.
* All instructions execute under the **effective CAP state** latched at the instruction boundary.
* All masking, predication, and fault behavior is identical to base RSV semantics.

No profile may override, reinterpret, or bypass CAP or XMEM rules.

---

### **1.4 Profile Summary**

This document currently defines the following profiles:

* **XRSV-PRMT** — Permutation, interleave/de-interleave, and table lookup operations.
* **XRSV-CEXP** — Masked compaction/expansion and compact scatter/gather patterns.
* **XRSVS** — Saturating and widening/narrowing integer arithmetic (advanced profile).

Profiles are independent of one another.
An implementation supporting one profile is not required to support any other.

---

### **1.5 Architectural Stability**

The profiles defined here are part of the **XPHMG v0.1 baseline**, but remain optional.

Future revisions of XPHMG may introduce additional RSV profiles or extend existing ones.
Such changes must preserve the architectural semantics of the RSV base and the guarantees defined by `XPHMG_CAP` and `XPHMG_XMEM`.

### 1.6 Encodings

These are *custom RSV opcodes* (custom-1 = `0101011`). They operate on integer or FP elements according to `EFF_PET/EFF_EW` in `CAP.PREC.STAT`.

We divide by **funct3** (major group) and then by **funct7** (sub-op). Unless noted, operands are **R-type**: `rd, rs1, rs2`.

#### 1.6.1 funct3 = `000` — PRMT (permute/zip/unzip/extract)

| funct7  | Mnemonic      | Type   | Operands                    | Notes                                                               |
| ------- | ------------- | ------ | --------------------------- | ------------------------------------------------------------------- |
| 0000000 | svperm.v      | R      | rd, rs1=a, rs2=idx          | rd[i] = a[idx[i]] (OOB: zero if ZMODE=1 else merge)                 |
| 0000001 | svpermi2.v    | R      | rd, rs1=a, rs2=idx          | Two-source via MSB of idx selects a/b; **b** is SVSRCB window       |
| 0000010 | svshuffle.v   | I      | rd, rs1=a, imm12=pat        | Structured patterns (see table below)                               |
| 0000011 | svzip.lo      | R-pair | rd(lo), rs1=a, rs2=b        | Interleave low; writes rd and rd^1                                  |
| 0000100 | svzip.hi      | R-pair | rd(lo), rs1=a, rs2=b        | Interleave high                                                     |
| 0000101 | svunzip.lo    | R-pair | rd(lo), rs1=a, rs2=b        | De-interleave low                                                   |
| 0000110 | svunzip.hi    | R-pair | rd(lo), rs1=a, rs2=b        | De-interleave high                                                  |
| 0000111 | svext.v       | I      | rd, rs1=a, imm12=lane_off   | Extract from `a || b` where **b** is SVSRCB window                  |

*R-pair legality:* Illegal if `rd` is odd or `SVDST.stride != 1`.

**Predication & ZMODE:** Masked-off lanes behave per `ZMODE` (merge or zero).

**Indices/OOB:** For `svperm.*` and `svtbl.*`, out-of-range indices produce **zero** if `ZMODE=1`, else **merge** keeps destination (portable choice). Implementations may set a *non-architectural* OOB counter.

**`svshuffle.v` pattern encoding (imm12=pat):**
* Upper imm bits reserved (0).
* `000`: even/odd interleave (a0,b0,a1,b1,…)
* `001`: swap 2-lane groups (similar to “zip1/zip2” style)
* `010`: swap 4-lane groups
* `011`: reverse within 2-lane groups
* `100`: reverse within 4-lane groups
* `101`: full lane reverse
* `110..111`: reserved (impl-def)

#### 1.6.2 funct3 = `001` — TBL/TBX (table lookup across RSV windows)

| funct7  | Mnemonic  | Type | Operands              | Notes                                                                  |
| ------- | --------- | ---- | --------------------- | ---------------------------------------------------------------------- |
| 0000000 | `svtbl.v` | R    | `rd, rs1=t0, rs2=idx` | Table starts at `rs1` and spans the `SVSRCA` window; VL/EW define span |
| 0000001 | `svtbx.v` | R    | `rd, rs1=t0, rs2=idx` | Lookup or keep: OOB → keep `rd[i]`                                     |

*R-pair legality:* Illegal if `rd` is odd or `SVDST.stride != 1`.

> Implementation picks how many adjacent table regs are consumed (min 2). OOB rules as for `svperm.v`.

### 1.4 Examples

**A) Classic “unpack low/high” (interleave 32-bit):**

```asm
# ra = [a0 a1 a2 a3], rb = [b0 b1 b2 b3]
# Resultados: rd0(low), rd1(high) (pair-dest)
svon.one
svzip.lo rd0, ra, rb       # rd0=[a0 b0 a1 b1], rd1=[a2 b2 a3 b3]
```

**B) Byte-table lookup (8-bit elements) with fallback (tbx):**

```asm
# idx in rI; table starts at t0 and spans the SVSRCA window
svsetvl x0, 16
svon.one
svtbx.v rd, t0, rI         # in-range → LUT; OOB → keep rd (merge) or zero (ZMODE=1)
```

---

## 2 Profile **XRSV-CEXP** — Compact / Expand by Mask (and Scatter-Compact)

### 2.1 Overview

Adds masked **compress/expand** operations commonly required in data-parallel filtering, visibility processing, stream compaction, and sparse workload handling.

The profile also defines a memory-oriented scatter-compact and gather-expand mechanism suitable for contiguous output generation, operating fully under `XPHMG_XMEM` control and optional fault-only-first (FOF) behavior.

### 2.2 Encodings

| Mnemonic                              | Summary                                                                                                                  |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `svcompress rd, ra, pmask`            | Packs lanes of `ra` where `pmask[i]=1` into `rd` contiguously from lane 0; masked-off lanes at tail are don’t-care/zero. |
| `svexpand   rd, ra, pmask`            | Inverse: spreads contiguous elements from `ra` into `rd` at lanes where mask=1; else zero/merge per ZMODE.               |
| `svscat.cmp [base], ra, pmask, flags` | **Scatter-compact to memory**: survivors stored densely from `[base]` upward. `flags` may include domain/QoS hints.      |
| `svgath.exp ra, [base], pmask, flags` | **Gather-expand from memory**: reads contiguous region into the lanes where mask=1.                                      |

**Mask operand:** All `svcompress/svexpand/svscat.cmp/svgath.exp` consume the current `SVPMASK*`. The mask is **implicit** in the encoding (no extra register field).

**Notes**

* `pmask` is the current `SVPMASK*`; the explicit operand is for mnemonic clarity (encoding may not need it).
* `svscat.cmp` and `svgath.exp` consume `CAP.XMEM.DOM` and hints; they may also use a small immediate `sprof_id` to pick a streaming profile.

**Counts:** Many algorithms need the count of survivors. Provide:

* `svpopcnt.pm -> rx` : returns population count of current predicate (0..VL). (Can be encoded as a CSR read of a virtual counter if preferred.)


#### 2.2.1 funct3 = `010` — CEXP (register compact/expand + popcnt)

| funct7  | Mnemonic      | Type | Operands                 | Notes                                                      |
| ------- | ------------- | ---- | ------------------------ | ---------------------------------------------------------- |
| 0000000 | `svcompress`  | R    | `rd, rs1=a, rs2=ignored` | Packs lanes where predicate=1, dense from lane 0           |
| 0000001 | `svexpand`    | R    | `rd, rs1=a, rs2=ignored` | Spreads `a` into lanes where predicate=1; others per ZMODE |
| 0000010 | `svpopcnt.pm` | R    | `rd, rs1=x0, rs2=x0`     | `rd ← popcount(predicate)` (0..VL)                         |

#### 2.2.2 funct3 = `011` — CEXP.MEM (scatter-compact / gather-expand)

**I-type**; `rs1=base`, `imm12` carries small hints.

| funct7  | Mnemonic     | Type | Operands                     | imm12 layout                                               |
| ------- | ------------ | ---- | ---------------------------- | ---------------------------------------------------------- |
| 0000000 | `svscat.cmp` | I    | `rd=x0, rs1=base, imm12=cfg` | `[1:0] sprof_id`, `[2] nt_hint`, `[3] lock_hint`, others 0 |
| 0000001 | `svgath.exp` | I    | `rd=x0, rs1=base, imm12=cfg` | same                                                       |

**Semantics:** uses `CAP.XMEM.DOM` and the sprof id in imm to pick a streaming profile when `STREAM_PROF=1`. Honors FOF if enabled in `CAP.XMEM.SVMSCL`.
**FOF retry point:** on first fault, the element index is written to `SVFAULTI` and the operation aborts; software may retry from that index. Streaming hints derive from `imm12` and `CAP.XMEM.SPROF*`.

### 2.3 Examples

**A) Masked stream compaction in registers**

```asm
# ra holds values; SVPMASK* marks survivors
svon.one
svcompress rd, ra, SVPMASK
# rd now contains survivors densely packed from lane 0
```

**B) Write survivors contiguously to memory (scatter-compact)**

```asm
# base in rB; domain/profile by CAP.XMEM.*
svon.one
svscat.cmp [rB], ra, SVPMASK, flags=0
```

---

## 3 Profile **XRSVS** — Saturating & Widening/Narrowing Integer Operations

### 3.1 Overview

Adds direct saturating arithmetic and widening/narrowing integer variants commonly required in signal processing, graphics pipelines, and fixed-point arithmetic workflows.

All policies related to signedness, saturation, and narrowing behavior respect the effective `CAP.PREC.MODE` and `CAP.PREC.STAT` state.
Explicit opcodes remove ambiguity and simplify code generation while preserving deterministic behavior.

### 3.2 Encodings

| Mnemonic                                 | Summary                                                                   |
| ---------------------------------------- | ------------------------------------------------------------------------- |
| `svadd.sat.s/u   ra, rb -> rd`           | Per-lane saturating add (signed/unsigned).                                |
| `svsub.sat.s/u   ra, rb -> rd`           | Per-lane saturating sub (signed/unsigned).                                |
| `svavg.rnd.s/u   ra, rb -> rd`           | Rounded average (optionally saturating on overflow).                      |
| `svabs.sat.s     ra -> rd`               | Saturating absolute (minint clamps).                                      |
| `svmul.wide.s/u  ra, rb -> rd_hi:rd_lo`  | Widening multiply: EW×EW → 2×EW (returns hi:lo or only lo if `rd_hi`=x0). |
| `svmla.wide.s/u  acc_hi:acc_lo, ra, rb`  | Widening multiply-accumulate into 2×EW accumulator pair.                  |
| `svnarrow.sat.s/u ra_hi:ra_lo -> rd`     | Narrow with saturation: 2×EW → EW.                                        |
| `svshift.wide.s  ra, imm -> rd_hi:rd_lo` | Arithmetic wide shift (2×EW accumulator).                                 |
| `svmin.sv/svmax.sv.s/u ra, rb -> rd`     | Per-lane min/max (signed/unsigned).                                       |

**Type selection & element width**

* Element type/width come from `CAP.PREC.STAT.{EFF_PET,EFF_EW}`.
* Signed vs unsigned is explicit (`.s`/`.u`) or by `CAP.PREC.MODE.UNS` when not annotated (implementation choice; recommend explicit mnemonics).

**Saturation semantics**

* On overflow, clamp to representable min/max of the **destination** width.
* Set `CAP.PREC.STAT.SAT_HIT`. Traps are not raised (this is value-saturating arithmetic, not exception-driven).

**Widen/narrow rules**

* `svmul.wide`/`svmla.wide` produce a full-precision 2×EW product/accumulator.
* `svnarrow.sat` narrows back with saturation (not rounding; if you want rounding, add a `.rnd` variant).

#### 3.2.1 funct3 = `100` — XRSVS (saturating arithmetic, min/max, avg)

| funct7  | Mnemonic      | Type | Operands       |
| ------- | ------------- | ---- | -------------- |
| 0000000 | `svadd.sat.s` | R    | `rd, rs1, rs2` |
| 0000001 | `svadd.sat.u` | R    | `rd, rs1, rs2` |
| 0000010 | `svsub.sat.s` | R    | `rd, rs1, rs2` |
| 0000011 | `svsub.sat.u` | R    | `rd, rs1, rs2` |
| 0000100 | `svabs.sat.s` | R    | `rd, rs1, x0`  |
| 0000101 | `svavg.rnd.s` | R    | `rd, rs1, rs2` |
| 0000110 | `svavg.rnd.u` | R    | `rd, rs1, rs2` |
| 0000111 | `svmin.s`     | R    | `rd, rs1, rs2` |
| 0001000 | `svmin.u`     | R    | `rd, rs1, rs2` |
| 0001001 | `svmax.s`     | R    | `rd, rs1, rs2` |
| 0001010 | `svmax.u`     | R    | `rd, rs1, rs2` |

Saturation sets `CAP.PREC.STAT.SAT_HIT`. No traps.

#### 3.2.2 funct3 = `101` — XRSVS.W (widening multiply / MAC / narrow / wide-shift)

| funct7  | Mnemonic         | Type   | Operands                | Pair                   |
| ------- | ---------------- | ------ | ----------------------- | ---------------------- |
| 0000000 | `svmul.wide.s`   | R-pair | `rd(lo), rs1, rs2`      | yes                    |
| 0000001 | `svmul.wide.u`   | R-pair | `rd(lo), rs1, rs2`      | yes                    |
| 0000010 | `svmla.wide.s`   | R-pair | `rd(lo), rs1, rs2`      | yes (acc += a*b)       |
| 0000011 | `svmla.wide.u`   | R-pair | `rd(lo), rs1, rs2`      | yes                    |
| 0000100 | `svnarrow.sat.s` | R      | `rd, rs1=lo, rs2=hi`    | n/a                    |
| 0000101 | `svnarrow.sat.u` | R      | `rd, rs1=lo, rs2=hi`    | n/a                    |
| 0000110 | `svshift.wide.s` | I-pair | `rd(lo), rs1=lo, imm12` | yes (arith shift 2×EW) |

**Accumulator use for `svmla.wide.*`:** `rd:rd^1` pre-holds accumulator; result is written back to the same pair.
*Pair-dest legality:* Illegal if `rd` is odd or `SVDST.stride != 1`. Accumulators for `svmla.wide.*` are preloaded in `rd:rd^1`.

### 3.3 Examples

**A) INT8 → INT16 widening MAC with final saturating narrow to INT8**

```asm
# acc_hi:acc_lo hold 16-bit accumulators, ra/rb are INT8 lanes
svsetvl x0, 16
svon.one
svmla.wide.su acc_hi:acc_lo, ra, rb   # signed×unsigned → 16-bit accum

# ... later, narrow back to INT8 with saturation:
svon.one
svnarrow.sat.su acc_hi:acc_lo -> rd
```

**B) Saturating absolute & average**

```asm
svon.one
svabs.sat.s ra -> rd
svavg.rnd.u ra, rb -> rd
```

---

## 4 Mask, Exceptions, and Policy Interactions (All Profiles)

* **Masks:** All instructions honor `SVPMASK*`. Lane writeback follows `CAP.PREC.MODE.ZMODE` (merge vs zeroing).
* **Numeric policy:** FP ops (if any permutation acts on FP payloads) obey `CAP.PREC.MODE.{FP_RMODE,SAE_DEF,NAN_POL,FTZ}` and `svon.fpctl` overrides. Integer saturating ops are **value-saturating** and do **not** raise traps; implementations set `SAT_HIT`.
* **FOF:** Not applicable to pure register permutes/compactions; applies only to memory-touching forms (`svscat.cmp`, `svgath.exp`) via `CAP.XMEM.SVM*` (and the standard RSV gather/scatter FOF behavior).
* **Determinism:** Effective exception-enable view is mirrored in `CAP.PREC.STAT.IE_MASK` per your base spec; these profiles do not alter that rule.

---

## 5 Minimal Conformance Sets

To keep adoption incremental, define three **levels** per profile:

* **XRSV-PRMT-M1 (Minimum):** `svzip.{lo,hi}`, `svunzip.{lo,hi}`, `svext.v`, `svshuffle.v` with patterns {000,101}.

* **XRSV-PRMT-M2 (Plus):** add `svperm.v`, `svpermi2.v`.

* **XRSV-PRMT-F (Full):** add `svtbl.v`, `svtbx.v`.

* **XRSV-CEXP-M1:** `svcompress`, `svexpand`, `svpopcnt.pm`.

* **XRSV-CEXP-F:** add `svscat.cmp`, `svgath.exp` (memory forms).

* **XRSVS-M1:** `svadd.sat.*`, `svsub.sat.*`, `svabs.sat.s`.

* **XRSVS-M2:** add `svmul.wide.*`, `svmla.wide.*`, `svnarrow.sat.*`.

* **XRSVS-F:** add `svavg.rnd.*`, `svmin.*`, `svmax.*`, `svshift.wide.s`.

Each level MUST list supported `EFF_EW` (e.g., 8/16/32). Unsupported widths: **illegal instruction**.

---

## 6 Conventions

* **Major opcode:** `custom-1` (`0101011`) for all XRSV-Profiles instructions.
* **Types used:** R-type (register), I-type (small immediates), and “R-pair” (R-type with an implied second destination via RSV stride).
* **Pair-dest rule (widening ops):** when `PAIR=1` (see funct7), results are written to **rd(lo)** and **rd^1 (hi)**. This is **legal only if** `rd` is even **and** `SVDST.stride==1`. Otherwise: *Illegal Instruction*.
* **Masks & ZMODE:** All ops honor `SVPMASK*`; masked writeback follows `CAP.PREC.MODE.ZMODE`.
* **Numeric policy:** FP/INT behavior per `CAP.PREC.*` (SAE/RC/NaN/FTZ), with optional `svon.fpctl` override.

### 6.1 Opcode map (custom-1 = `0101011`)

We divide by **funct3** (major group) and then by **funct7** (sub-op). Unless noted, operands are **R-type**: `rd, rs1, rs2`.
**funct3:**
* `000 → PRMT`;
* `001 → TBL/TBX`;
* `010 → CEXP(reg)`;
* `011 → CEXP.MEM`;
* `100 → XRSVS`;
* `101 → XRSVS.W`;
* `110/111 → reserved`.

### 6.2 funct3 = `110` / `111` — Reserved

* Keep for future (e.g., FP permutes, bit-matrix ops, polynomial CRC, etc.). Must raise *Illegal Instruction* if not implemented.

---
## **7. Cross-Domain Guarantees for RSV-Profiles (Normative)**

This section is added as a new normative block near the end of `xphmg_rsv-profiles.md`.

### **7.1 CAP Precision & Mask Semantics**

All RSV-Profile instructions (PRMT, CEXP, SAT) operate on values interpreted using:

* `EFF_PET`, `EFF_EW`
* `EFF_ACCW`
* `EFF_FP_RMODE`
* `EFF_ZMODE`
* `EFF_Q`, ZP, SCALE_SH
* ZMODE zero-vs-merge

Profiles **must not override** global precision behavior.
When narrowing, if the target format is not natively supported, the hardware must:

* perform a legal fallback,
* and set `DOWNCAST_TAKEN` or `UNSUP_FMT`.

### **7.2 XMEM Coherence & Gather/Scatter Rules**

Any profile that interacts with memory (compaction/gather expansion) must:

* use `CAP.XMEM.SVM*` definitions,
* obey RSV predicates,
* obey ZMODE,
* apply FOF identically to RSV and MTX domains,
* follow coherence domain from `CAP.XMEM.DOM`.

### **7.3 Saturating Profiles & Sticky Flags**

All saturating ops must:

* saturate according to the representable range of `EFF_PET/EW`,
* set `SAT_HIT`,
* treat masked stores using ZMODE,
* not change global exception routing.

### **7.4 Permute Profiles & AOS↔SOA Rules**

All permutation-based profiles must:

* preserve FP bit patterns,
* use the same narrowing/widening semantics as MTX,
* never bypass CAP precision rules,
* use ZMODE for masked destinations,
* update STAT if unsupported narrowing is requested.

### **7.5 Interoperability With Other Domains**

RSV-Profiles are required to produce data layouts acceptable to:

* MTX 2×2..4×4 tiles,
* GFX UV/RGBA/packed integer formats,
* RT node formats (e.g., NODE4/NODE8 barycentric and bounds layout).

Profiles must not introduce domain-specific behavior; the software should assume full interchangeability.

---

## 8 Operand & Immediate Diagrams

**R-type (PRMT/CEXP/SAT):**

```
| funct7 | rs2 | rs1 | funct3 | rd | 0101011 |
  [31:25] [24:20] [19:15] [14:12] [11:7] [6:0]
```

**I-type (shuffle/ext/scatter/gather/shift):**

```
| imm[11:0] | rs1 | funct3 | rd | 0101011 |
  [31:20]   [19:15][14:12] [11:7][6:0]
```

* `svshuffle.v`: `imm[2:0]=pattern`, rest 0
* `svext.v`:     `imm[11:0]=lane_offset` (u12)
* `svscat.cmp` / `svgath.exp`: `imm[1:0]=sprof_id`, `imm[2]=nt_hint`, `imm[3]=lock_hint`, others 0
* `svshift.wide.s`: `imm[4:0]=shift_amount` (for EW≤32; implementations may extend)

---

## 9 Programmer Hints & Mapping Guidance

This section provides **informative mapping examples** between common scalar-vector idioms and the corresponding RSV-Profiles instructions.

The mappings are intended to assist programmers and compiler authors in recognizing familiar patterns.
They are **non-normative** and do not imply compatibility with, or derivation from, any external ISA.

Examples include:

* Permutation and shuffle idioms expressed using `svperm.v` and `svshuffle.v`
* Interleave and de-interleave patterns using `svzip.*` / `svunzip.*`
* Mask-based compaction and expansion using `svcompress` / `svexpand`
* Widening multiply-accumulate and saturating arithmetic using XRSVS instructions

These examples illustrate **conceptual equivalence only** and must not be interpreted as binary or semantic compatibility guarantees.

---

## 10 Exceptions & Debug

* No new sticky flags required beyond `SAT_HIT` you already expose in `CAP.PREC.STAT`.
* Optional PMU/debug counters (non-architectural): `perm_oob`, `tbl_oob`, `cmp_survivors`, `cmp_bytes`, etc.

---
*Licensed under CC-BY-SA 4.0 — Open Specification maintained by Pedro H. M. Garcia.*

Designed as extension for `XPHMG_RSV` and for integration with `XPHMG_CAP`, and `XPHMG_XMEM`.
