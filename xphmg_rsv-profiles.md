# **XPHMG_RSV-Profiles**

## Optional Scalar-Vector Profiles for XPHMG_RSV

**Category:** Vendor Extension Addendum (`XPHMG_RSV-Profiles`)
**Version:** 0.1.1
**Author:** Pedro H. M. Garcia
**License:** CC-BY-SA 4.0 (open specification)

---

## **1. Overview (Informative)**

`XPHMG_RSV-Profiles` defines a set of **optional instruction profiles** that extend the base `XPHMG_RSV` (Scalar-Vectors) execution model.

These profiles introduce commonly needed scalar-vector operations (such as permutation, compaction/expansion, and saturating or widening arithmetic) without altering the architectural contract of RSV or the authority of `XPHMG_CAP` and `XPHMG_XMEM`.

Support for RSV profiles is **entirely optional**.
An implementation may support none, some, or all profiles independently.
If a profile-specific instruction is not supported, it must trap as an *illegal instruction*.

### **1.1 Relationship to XPHMG_RSV**

The base `XPHMG_RSV` extension defines:

* scalar-vector reinterpretation,
* vector length and register windows,
* predication semantics,
* interaction with CAP-controlled numeric policy and XMEM-controlled memory behavior.

`XPHMG_RSV-Profiles` **does not modify** any of these rules.

All profiles:

* reuse the existing RSV enablement (`svon.*`),
* operate on the same register windows and state (`SVSTATE`, `SVSRC*`, `SVDST`),
* inherit numeric precision, rounding, saturation, quantization, and exception handling from `XPHMG_CAP`,
* inherit memory behavior, domains, and gather/scatter semantics from `XPHMG_XMEM`.

No profile introduces new global CSRs, new numeric modes, or profile-specific memory rules.

### **1.2 Purpose of Profiles**

The profiles defined in this document serve three main purposes:

1. **Encapsulation of common idioms**
   Frequently used scalar-vector patterns (e.g., permutation, compaction, widening arithmetic) are grouped into named profiles rather than being folded into the RSV base.

2. **Incremental implementation**
   Implementers may choose to support only the RSV base, or selectively add profiles according to target workload, silicon budget, or performance goals.

3. **Architectural clarity**
   By isolating higher-level operations into optional profiles, the RSV base remains compact, predictable, and easy to reason about.

Profiles are therefore **extensions of convenience**, not architectural requirements.

### **1.3 Normative Scope**

Unless explicitly stated otherwise:

* All instructions defined in this document are **optional**.
* Lack of support must result in an *illegal instruction* exception.
* All instructions execute under the **effective CAP state** latched at the instruction boundary.
* Predication for all RSV profiles consumes the unified XPHMG predication model; the effective predicate is sourced exclusively from PMASK as defined in `xphmg.md`, and profiles do not define, enable, disable, or modify predication behavior.
* All fault behavior is identical to base RSV semantics.

No profile may override, reinterpret, or bypass CAP or XMEM rules.

### **1.4 Profile Catalog**

This document currently defines the following profiles:

* **XRSV-PRMT** - Permutation, interleave/de-interleave, and table lookup operations.
* **XRSV-CEXP** - Masked compaction/expansion and compact scatter/gather patterns.
* **XRSVS** - Saturating and widening/narrowing integer arithmetic (advanced profile).

Profiles are independent of one another.
An implementation supporting one profile is not required to support any other.

### **1.5 Architectural Stability**

The profiles defined here are part of the **XPHMG v0.1 baseline**, but remain optional.

Future revisions of XPHMG may introduce additional RSV profiles or extend existing ones.
Such changes must preserve the architectural semantics of the RSV base and the guarantees defined by `XPHMG_CAP` and `XPHMG_XMEM`.

### 1.6 Conceptual Model

* Profiles are **execution subsets** layered on top of the RSV base. Each profile contributes only additional opcodes; it does not redefine RSV state, CSRs, or enablement flows.
* All profiles **share the RSV execution context**: `SVSTATE`, `SVSRC*`, `SVDST`, and the VL/EW model defined by `XPHMG_RSV`.
* A profile is either **absent or fully present** in an implementation. Partial support within a named profile is illegal unless explicitly enumerated in the conformance levels of Section 7.
* Unsupported instructions are architecturally required to raise *illegal instruction*, mirroring RSV base behavior for absent subfeatures.

### 1.7 Architectural Guarantees & Invariants

* Profiles **cannot override** `XPHMG_CAP` numeric policy (precision, rounding, saturation, quantization, NaN/FTZ handling) or `XPHMG_XMEM` memory policy (domains, FOF, streaming hints).
* All profile instructions execute under the **effective CAP state** latched at the instruction boundary; there is no profile-local policy override.
* **Register windowing and pair-destination legality** are inherited from RSV: pair writes require even `rd` and `SVDST.stride == 1`; violations are *illegal instruction*.
* Profiles do **not introduce new CSRs**; any profile-related observability (e.g., counters) is non-architectural unless defined elsewhere.

### 1.8 Interaction with CAP / XMEM / Other Extensions

* `XPHMG_CAP` supplies element type/width (`EFF_PET/EFF_EW`), rounding, saturation flagging (`SAT_HIT`), and exception masking; profiles must not alter these mechanisms.
* `XPHMG_XMEM` governs all memory effects of profile instructions that touch memory (e.g., compact scatter/gather). Fault-only-first and streaming behaviors are inherited unchanged.
* RSV enable/disable sequences (`svon.*`) apply uniformly; profiles do not add alternate enablement or power-state semantics.
* Data produced by profiles is intended to interoperate with MTX/GFX/RT/ACCEL domains without requiring reformatting beyond what RSV already guarantees.

### 1.9 Undefined / Reserved Behavior

* Execution of a profile instruction when the profile is not implemented: *illegal instruction*.
* Encodings not listed for a given profile or reserved funct3/funct7 combinations: *illegal instruction*.
* Profile-local configuration CSRs: **reserved** in v0.1; any future addition must not violate the invariants above.

### 1.10 Notes for Implementers (Informative)

* Optional microarchitectural counters (e.g., permutation OOB counts) are allowed but are non-architectural unless separately standardized.
* Toolchains should treat profile availability as a separate feature bit per profile; absence must default to software emulation or feature gating.

---

## 2 Architectural Context & Common Rules (Normative)

This section records the common rules inherited from `XPHMG_RSV`, `XPHMG_CAP`, and `XPHMG_XMEM` that apply to **all** RSV-Profile instructions. Profiles are additive to the RSV execution model and rely on the same architectural contract for enablement, masking, numeric policy, memory domains, and exception visibility.

### 2.1 Mask, Exceptions, and Policy Interactions (All Profiles)

Predication for all RSV-Profile instructions follows the unified XPHMG predication model defined in `xphmg.md`.

* **CAP state latching:** The effective CAP state (`EFF_*`) is latched at the instruction boundary exactly as in RSV. Profiles do not establish profile-local overrides; numeric behavior (rounding, NaN, FTZ, saturation flags) is wholly defined by CAP.
* **Numeric policy:** FP payload permutations preserve bit patterns and observe CAP FP policy; integer saturating ops are value-saturating only and set `SAT_HIT` without raising traps.
* **FOF and retry:** Register-only forms cannot generate memory faults. Memory-touching forms (`svscat.cmp`, `svgath.exp`) inherit the RSV/`XPHMG_XMEM` fault-only-first rules, including recording the first-fault index and terminating the operation per the base retry contract.
* **Determinism and exceptions:** Exception enables/flags are visible through `CAP.PREC.STAT` as in RSV. Profiles do not add alternate exception routing or stickiness.
* **Illegal instruction:** Violations of pair-destination legality, reserved encodings, or unsupported profiles are architecturally illegal and must trap as *illegal instruction*.

### 2.2 Cross-Domain Guarantees for RSV-Profiles (Normative)

NOTE: Relocated from former Section 7; content unchanged.

#### 2.2.1 CAP Precision & Mask Semantics

All RSV-Profile instructions (PRMT, CEXP, SAT) operate on values interpreted using:

* `EFF_PET`, `EFF_EW`
* `EFF_ACCW`
* `EFF_FP_RMODE`
* `EFF_ZMODE`
* `EFF_Q`, ZP, SCALE_SH

Profiles **must not override** global precision behavior.
When narrowing, if the target format is not natively supported, the hardware must:

* perform a legal fallback,
* and set `DOWNCAST_TAKEN` or `UNSUP_FMT`.

#### 2.2.2 XMEM Coherence & Gather/Scatter Rules

Any profile that interacts with memory (compaction/gather expansion) must:

* use `CAP.XMEM.SVM*` definitions,
* obey effective predication as defined by the XPHMG predication model (`xphmg.md`),
* apply FOF identically to RSV and MTX domains,
* follow coherence domain from `CAP.XMEM.DOM`.

#### 2.2.3 Saturating Profiles & Sticky Flags

All saturating ops must:

* saturate according to the representable range of `EFF_PET/EW`,
* set `SAT_HIT`,
* treat masked stores per the XPHMG predication model (`xphmg.md`),
* not change global exception routing.

#### 2.2.4 Permute Profiles & AOS/SOA Rules

All permutation-based profiles must:

* preserve FP bit patterns,
* use the same narrowing/widening semantics as MTX,
* never bypass CAP precision rules,
* update STAT if unsupported narrowing is requested.

#### 2.2.5 Interoperability With Other Domains

RSV-Profiles are required to produce data layouts acceptable to:

* MTX 2x2..4x4 tiles,
* GFX UV/RGBA/packed integer formats,
* RT node formats (e.g., NODE4/NODE8 barycentric and bounds layout).

Profiles must not introduce domain-specific behavior; the software should assume full interchangeability.

### 2.3 Notes for Implementers (Informative)

* Non-architectural observability (e.g., OOB counters for permutation/table) is permitted but must not alter architected state or timing guarantees.
* Streaming hints embedded in profile instructions must be interpreted using the same mechanisms as RSV/`XPHMG_XMEM`; if a hint is unrecognized, it is treated as reserved/ignored without altering architectural behavior.
* Toolchains should assume a single predicate/mask domain shared across all profiles; no profile-specific predicate registers exist in v0.1.

---

## 3 Encoding Conventions (Normative)

This section summarizes the common encoding conventions for all RSV-Profile instructions. It does not redefine opcode ownership or add new encoding space beyond the RSV custom-1 allocation already assumed by the base extension.

### 3.1 Major Opcode & funct3 Map

* **Major opcode:** `custom-1` (`0101011`) for all XRSV-Profiles instructions.
* **Types used:** R-type (register), I-type (small immediates), and "R-pair" (R-type with an implied second destination via RSV stride).
* **Pair-dest rule (widening ops):** when `PAIR=1` (see funct7), results are written to **rd(lo)** and **rd^1 (hi)**. This is **legal only if** `rd` is even **and** `SVDST.stride==1`. Otherwise: *Illegal Instruction* (per Section 2).
* **Predication:** All ops consume the effective predicate defined by the XPHMG predication model (see `xphmg.md`).
* **Numeric policy:** FP/INT behavior per `CAP.PREC.*` (SAE/RC/NaN/FTZ), with optional `svon.fpctl` override. No profile-specific numeric policy exists.
* **Reserved bit patterns:** Unless specified, unused immediate bits and reserved funct7 values must be encoded as zero and are architecturally *illegal instruction* if non-zero.

**funct3:**
* `000` - PRMT;
* `001` - TBL/TBX;
* `010` - CEXP(reg);
* `011` - CEXP.MEM;
* `100` - XRSVS;
* `101` - XRSVS.W;
* `110/111` - reserved.

### 3.2 Operand & Immediate Formats

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
* `svshift.wide.s`: `imm[4:0]=shift_amount` (for EW<=32; implementations may extend)

**Immediate treatment and sign rules**

* All immediates are zero-extended unless otherwise stated by the base encoding class (I-type sign-extends the full 12-bit field in the usual RISC-V manner when consumed as signed).
* Reserved immediate bits are **required to be zero** in v0.1; non-zero encodings are *illegal instruction*.
* Implementations may decode additional imm bits internally for microarchitectural use, but such uses are non-architectural and must not change architected behavior.

### 3.3 Reserved Encodings

* **funct3 = `110` / `111`:** Reserved (e.g., FP permutes, bit-matrix ops, polynomial CRC, etc.). Must raise *Illegal Instruction* if not implemented.

### 3.4 Notes for Implementers (Informative)

* Encodings that overlap with future profiles must remain *illegal instruction* in v0.1; do not alias them to implementation-defined behavior.
* Pair-destination legality checks should occur before state updates to preserve determinism and fault-only-first behavior of memory forms.
* Toolchains should model availability at the funct3-major level (PRMT, TBL/TBX, CEXP, XRSVS) with finer-grained conformance levels defined in Section 7.

---

## 4 Profile **XRSV-PRMT** - Permutation/Table (Normative)

### 4.1 Profile Overview

**XRSV-PRMT** defines permutation, interleave/de-interleave, and table lookup operations over RSV register windows. The profile reuses the RSV execution context (`SVSTATE`, `SVSRC*`, `SVDST`) and does not introduce profile-local state. Its purpose is to provide common data-reordering idioms without changing RSV's masking, numeric, or memory contracts.

Conceptual model:

* All PRMT operations are lane-wise, using indices or structured patterns to select source elements from the active RSV windows.
* Pair destinations follow the RSV pair rule (even `rd`, `SVDST.stride==1`), writing low and high halves into adjacent registers.
* Table lookups (`svtbl.v`/`svtbx.v`) treat the RSV window as a contiguous table; the span is defined by VL and element width.

### 4.2 Instruction List & Encodings

These are *custom RSV opcodes* (custom-1 = `0101011`). They operate on integer or FP elements according to `EFF_PET/EFF_EW` in `CAP.PREC.STAT`.

We divide by **funct3** (major group) and then by **funct7** (sub-op). Unless noted, operands are **R-type**: `rd, rs1, rs2`.

#### 4.2.1 funct3 = `000` - PRMT (permute/zip/unzip/extract)

| funct7  | Mnemonic      | Type   | Operands                    | Notes                                                               |
| ------- | ------------- | ------ | --------------------------- | ------------------------------------------------------------------- |
| 0000000 | svperm.v      | R      | rd, rs1=a, rs2=idx          | rd[i] = a[idx[i]] (OOB handled per RSV predicated writeback policy; see `xphmg.md`) |
| 0000001 | svpermi2.v    | R      | rd, rs1=a, rs2=idx          | Two-source via MSB of idx selects a/b; **b** is SVSRCB window       |
| 0000010 | svshuffle.v   | I      | rd, rs1=a, imm12=pat        | Structured patterns (see table below)                               |
| 0000011 | svzip.lo      | R-pair | rd(lo), rs1=a, rs2=b        | Interleave low; writes rd and rd^1                                  |
| 0000100 | svzip.hi      | R-pair | rd(lo), rs1=a, rs2=b        | Interleave high                                                     |
| 0000101 | svunzip.lo    | R-pair | rd(lo), rs1=a, rs2=b        | De-interleave low                                                   |
| 0000110 | svunzip.hi    | R-pair | rd(lo), rs1=a, rs2=b        | De-interleave high                                                  |
| 0000111 | svext.v       | I      | rd, rs1=a, imm12=lane_off   | Extract from `a || b` where **b** is SVSRCB window                  |

*R-pair legality:* Illegal if `rd` is odd or `SVDST.stride != 1`.

**Indices/OOB:** For `svperm.*` and `svtbl.*`, out-of-range indices are handled per RSV predicated writeback policy (see `xphmg.md`). Implementations may set a *non-architectural* OOB counter.

**`svshuffle.v` pattern encoding (imm12=pat):**
* Upper imm bits reserved (0).
* `000`: even/odd interleave (a0,b0,a1,b1,...)
* `001`: swap 2-lane groups (similar to zip1/zip2 style)
* `010`: swap 4-lane groups
* `011`: reverse within 2-lane groups
* `100`: reverse within 4-lane groups
* `101`: full lane reverse
* `110..111`: reserved (impl-def)

#### 4.2.2 funct3 = `001` - TBL/TBX (table lookup across RSV windows)

| funct7  | Mnemonic  | Type | Operands              | Notes                                                                  |
| ------- | --------- | ---- | --------------------- | ---------------------------------------------------------------------- |
| 0000000 | `svtbl.v` | R    | `rd, rs1=t0, rs2=idx` | Table starts at `rs1` and spans the `SVSRCA` window; VL/EW define span |
| 0000001 | `svtbx.v` | R    | `rd, rs1=t0, rs2=idx` | Lookup or keep: OOB -> keep `rd[i]`                                    |

*R-pair legality:* Illegal if `rd` is odd or `SVDST.stride != 1`.

> Implementation picks how many adjacent table regs are consumed (min 2). OOB handling follows RSV predicated writeback policy (see `xphmg.md`).

### 4.3 Semantic Notes

* **Predication:** PRMT/TBL/TBX instructions consume the effective predicate defined by the XPHMG predication model (see `xphmg.md`).
* **Pair destinations:** `PAIR=1` encodings (zip/unzip) are legal only when `rd` is even and `SVDST.stride==1`; otherwise *illegal instruction*. Pair writes occur as in RSV widening ops: `rd` receives the low part, `rd^1` the high part.
* **Index interpretation:** Indices are lane-wise unsigned values. For `svperm.v`/`svpermi2.v`/`svtbl.v`/`svtbx.v`, an index selects an element within the active source window(s). `svpermi2.v` uses the MSB of the index to select between `SVSRCA`/`SVSRCB`.
* **Out-of-bounds (OOB) handling:** For indexed permutes and table lookups, OOB indices are handled per RSV predicated writeback policy (see `xphmg.md`). Implementations may count OOB events non-architecturally.
* **Structured patterns:** Reserved pattern encodings in `svshuffle.v` must be treated as *illegal instruction* if not implemented. Upper immediate bits are reserved and must be zero.
* **Table span:** `svtbl.v`/`svtbx.v` interpret the table as the contiguous RSV source window starting at `rs1`, spanning VL elements of width `EFF_EW`. Implementations may consume additional adjacent registers but must not read beyond the architecturally visible window defined by VL/EW.
* **FP payloads:** Permutations operating on FP elements preserve bit patterns; no NaN canonicalization or payload modification is performed by the permute itself. Numeric mode remains as latched by CAP.
* **Memory:** PRMT/TBL/TBX are register-only and do not generate memory accesses.

### 4.4 Notes for Implementers (Informative)

* For `svpermi2.v`, the choice of A/B source based on the index MSB should occur before bounds checking; an OOB after source selection follows the standard OOB rule above.
* Hardware may fuse `svtbl.v`/`svtbx.v` into wider internal table fetches, but architectural span is limited to the active RSV window; do not expose implementation-specific aliasing.
* Toolchains should advertise XRSV-PRMT availability separately from TBL/TBX conformance levels (see Section 7) when finer granularity is needed.

### 4.4 Examples

**A) Classic unpack low/high (interleave 32-bit):**

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
svtbx.v rd, t0, rI         # in-range -> LUT; OOB -> keep rd per RSV predicated writeback policy (see `xphmg.md`)
```

---

## 5 Profile **XRSV-CEXP** - Compact / Expand by Mask (and Scatter-Compact) (Normative)

### 5.1 Profile Overview

Adds masked **compress/expand** operations commonly required in data-parallel filtering, visibility processing, stream compaction, and sparse workload handling.

The profile also defines a memory-oriented scatter-compact and gather-expand mechanism suitable for contiguous output generation, operating fully under `XPHMG_XMEM` control and optional fault-only-first (FOF) behavior.

Conceptual model:

* Register forms (`svcompress`, `svexpand`, `svpopcnt.pm`) operate entirely in registers under the effective predicate, compacting or expanding surviving elements without changing VL or predicate state.
* Memory forms (`svscat.cmp`, `svgath.exp`) stream survivors contiguously to or from memory under `CAP.XMEM` policy, using the effective predicate to define survivors/slots and honoring FOF.
* All forms consume the effective predicate implicitly; no explicit predicate operand exists in the encoding.

### 5.2 Register Encodings & Semantics

| Mnemonic                              | Summary                                                                                                                  |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `svcompress rd, ra, pmask`            | Packs lanes of `ra` where `pmask[i]=1` into `rd` contiguously from lane 0; tail lanes follow base predicated writeback policy (see `xphmg.md`). |
| `svexpand   rd, ra, pmask`            | Inverse: spreads contiguous elements from `ra` into `rd` at lanes where mask=1; others follow base predicated writeback policy (see `xphmg.md`). |
| `svscat.cmp [base], ra, pmask, flags` | **Scatter-compact to memory**: survivors stored densely from `[base]` upward. `flags` may include domain/QoS hints.      |
| `svgath.exp ra, [base], pmask, flags` | **Gather-expand from memory**: reads contiguous region into the lanes where mask=1.                                      |

**Predicate operand:** All `svcompress/svexpand/svscat.cmp/svgath.exp` consume the current effective predicate from PMASK. The predicate is **implicit** in the encoding (no extra register field).

**Notes**

* `pmask` is the current effective predicate from PMASK; the explicit operand is for mnemonic clarity (encoding may not need it).
* `svscat.cmp` and `svgath.exp` consume `CAP.XMEM.DOM` and hints; they may also use a small immediate `sprof_id` to pick a streaming profile.

**Counts:** Many algorithms need the count of survivors. Provide:

* `svpopcnt.pm -> rx` : returns population count of current predicate (0..VL). (Can be encoded as a CSR read of a virtual counter if preferred.)

#### 5.2.1 funct3 = `010` - CEXP (register compact/expand + popcnt)

| funct7  | Mnemonic      | Type | Operands                 | Notes                                                      |
| ------- | ------------- | ---- | ------------------------ | ---------------------------------------------------------- |
| 0000000 | `svcompress`  | R    | `rd, rs1=a, rs2=ignored` | Packs lanes where predicate=1, dense from lane 0           |
| 0000001 | `svexpand`    | R    | `rd, rs1=a, rs2=ignored` | Spreads `a` into lanes where predicate=1; others per predicated writeback policy (see `xphmg.md`) |
| 0000010 | `svpopcnt.pm` | R    | `rd, rs1=x0, rs2=x0`     | `rd -> popcount(predicate)` (0..VL)                        |

### 5.3 Memory Encodings & Semantics (scatter-compact / gather-expand)

**I-type**; `rs1=base`, `imm12` carries small hints.

| funct7  | Mnemonic     | Type | Operands                     | imm12 layout                                               |
| ------- | ------------ | ---- | ---------------------------- | ---------------------------------------------------------- |
| 0000000 | `svscat.cmp` | I    | `rd=x0, rs1=base, imm12=cfg` | `[1:0] sprof_id`, `[2] nt_hint`, `[3] lock_hint`, others 0 |
| 0000001 | `svgath.exp` | I    | `rd=x0, rs1=base, imm12=cfg` | same                                                       |

**Semantics:** uses `CAP.XMEM.DOM` and the sprof id in imm to pick a streaming profile when `STREAM_PROF=1`. Honors FOF if enabled in `CAP.XMEM.SVMSCL`.
**FOF retry point:** on first fault, the element index is written to `SVFAULTI` and the operation aborts; software may retry from that index. Streaming hints derive from `imm12` and `CAP.XMEM.SPROF*`.

### 5.4 Examples

**A) Masked stream compaction in registers**

```asm
# ra holds values; effective predicate marks survivors
svon.one
svcompress rd, ra, pmask
# rd now contains survivors densely packed from lane 0
```

**B) Write survivors contiguously to memory (scatter-compact)**

```asm
# base in rB; domain/profile by CAP.XMEM.*
svon.one
svscat.cmp [rB], ra, pmask, flags=0
```

TODO-ELABORATE: Clarify FOF/retry details and survivor count reporting across register and memory forms.

### 5.5 Semantic Notes

* **Predicate use:** The effective predicate defines survivors. The predicate is implicit; there is no encoded predicate operand. Predicate state is not mutated by these operations.
* **Ordering and contiguity:** `svcompress` writes survivors starting at lane 0 in order of increasing lane index; lanes beyond the survivor count follow base predicated writeback policy (see `xphmg.md`). `svexpand` consumes a dense source array from `rs1` and places elements only where the predicate bit is 1; other lanes follow the same policy.
* **Population count:** `svpopcnt.pm` reports the popcount of the current predicate (0..VL) and does not modify predicate state. Return width/format follow the base RSV integer register rules.
* **FOF and memory side effects:** `svscat.cmp`/`svgath.exp` follow RSV/`XPHMG_XMEM` fault-only-first rules. On first fault, no further memory accesses are performed; `SVFAULTI` captures the failing element index for retry. Partial stores/loads after the first fault are architecturally forbidden.
* **Streaming profile selection:** `sprof_id` and hint bits (`nt_hint`, `lock_hint`) are interpreted per `CAP.XMEM.SPROF*`; unrecognized combinations are reserved and must behave as *illegal instruction* or ignore hints without altering architectural state (per base XMEM rules).
* **Addressing and domains:** Base address is in `rs1`; domain/ordering/QoS are derived from `CAP.XMEM.DOM` and hint bits. Element size and stride follow `EFF_EW` and RSV scatter/gather rules; no profile-specific stride exists.
* **Interaction with CAP:** Numeric behavior follows CAP; no additional saturation or rounding occurs in CEXP beyond what CAP defines for integer element moves.

### 5.6 Notes for Implementers (Informative)

* Implementations may use internal survivor-count generation to optimize `svcompress`/`svscat.cmp`; survivor count is not architecturally visible except via `svpopcnt.pm` or fault retry index.
* Memory forms may fuse address generation for contiguous survivors but must still honor predicate gating and FOF termination. Writes/reads beyond the first fault are prohibited.
* Toolchains should model `svpopcnt.pm` availability together with CEXP-M1 conformance; absence should be handled via software popcount of the effective predicate if needed.

---

## 6 Profile **XRSVS** - Saturating & Widening/Narrowing Integer Operations (Normative)

### 6.1 Profile Overview

Adds direct saturating arithmetic and widening/narrowing integer variants commonly required in signal processing, graphics pipelines, and fixed-point arithmetic workflows.

All policies related to signedness, saturation, and narrowing behavior respect the effective `CAP.PREC.MODE` and `CAP.PREC.STAT` state.
Explicit opcodes remove ambiguity and simplify code generation while preserving deterministic behavior.

Conceptual model:

* XRSVS provides lane-wise integer arithmetic that clamps on overflow, plus widening/narrowing paths that round-trip values between EW and 2xEW representations.
* Saturation is value-saturating only; it does not raise traps. `SAT_HIT` is the architectural indication of saturation.
* Widening ops produce a 2xEW result into a pair destination; narrowing ops consume a 2xEW pair and clamp back to EW.
* Signedness is explicit in mnemonics (`.s`/`.u`) unless otherwise defined by CAP’s UNS mode; no profile-local sign controls exist.

### 6.2 Arithmetic Encodings & Semantics

| Mnemonic                                 | Summary                                                                   |
| ---------------------------------------- | ------------------------------------------------------------------------- |
| `svadd.sat.s/u   ra, rb -> rd`           | Per-lane saturating add (signed/unsigned).                                |
| `svsub.sat.s/u   ra, rb -> rd`           | Per-lane saturating sub (signed/unsigned).                                |
| `svavg.rnd.s/u   ra, rb -> rd`           | Rounded average (optionally saturating on overflow).                      |
| `svabs.sat.s     ra -> rd`               | Saturating absolute (minint clamps).                                      |
| `svmul.wide.s/u  ra, rb -> rd_hi:rd_lo`  | Widening multiply: EW->2xEW (returns hi:lo or only lo if `rd_hi`=x0).     |
| `svmla.wide.s/u  acc_hi:acc_lo, ra, rb`  | Widening multiply-accumulate into 2xEW accumulator pair.                  |
| `svnarrow.sat.s/u ra_hi:ra_lo -> rd`     | Narrow with saturation: 2xEW -> EW.                                       |
| `svshift.wide.s  ra, imm -> rd_hi:rd_lo` | Arithmetic wide shift (2xEW accumulator).                                 |
| `svmin.sv/svmax.sv.s/u ra, rb -> rd`     | Per-lane min/max (signed/unsigned).                                       |

**Type selection & element width**

* Element type/width come from `CAP.PREC.STAT.{EFF_PET,EFF_EW}`.
* Signed vs unsigned is explicit (`.s`/`.u`) or by `CAP.PREC.MODE.UNS` when not annotated (implementation choice; recommend explicit mnemonics).

**Saturation semantics**

* On overflow, clamp to representable min/max of the **destination** width.
* Set `CAP.PREC.STAT.SAT_HIT`. Traps are not raised (this is value-saturating arithmetic, not exception-driven).

**Widen/narrow rules**

* `svmul.wide`/`svmla.wide` produce a full-precision 2xEW product/accumulator.
* `svnarrow.sat` narrows back with saturation (not rounding; if you want rounding, add a `.rnd` variant).

#### 6.2.1 funct3 = `100` - XRSVS (saturating arithmetic, min/max, avg)

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

### 6.3 Widen/Narrow Encodings & Semantics

#### 6.3.1 funct3 = `101` - XRSVS.W (widening multiply / MAC / narrow / wide-shift)

| funct7  | Mnemonic         | Type   | Operands                | Pair                   |
| ------- | ---------------- | ------ | ----------------------- | ---------------------- |
| 0000000 | `svmul.wide.s`   | R-pair | `rd(lo), rs1, rs2`      | yes                    |
| 0000001 | `svmul.wide.u`   | R-pair | `rd(lo), rs1, rs2`      | yes                    |
| 0000010 | `svmla.wide.s`   | R-pair | `rd(lo), rs1, rs2`      | yes (acc += a*b)       |
| 0000011 | `svmla.wide.u`   | R-pair | `rd(lo), rs1, rs2`      | yes                    |
| 0000100 | `svnarrow.sat.s` | R      | `rd, rs1=lo, rs2=hi`    | n/a                    |
| 0000101 | `svnarrow.sat.u` | R      | `rd, rs1=lo, rs2=hi`    | n/a                    |
| 0000110 | `svshift.wide.s` | I-pair | `rd(lo), rs1=lo, imm12` | yes (arith shift 2xEW) |

**Accumulator use for `svmla.wide.*`:** `rd:rd^1` pre-holds accumulator; result is written back to the same pair.
*Pair-dest legality:* Illegal if `rd` is odd or `SVDST.stride != 1`. Accumulators for `svmla.wide.*` are preloaded in `rd:rd^1`.

### 6.4 Examples

**A) INT8 -> INT16 widening MAC with final saturating narrow to INT8**

```asm
# acc_hi:acc_lo hold 16-bit accumulators, ra/rb are INT8 lanes
svsetvl x0, 16
svon.one
svmla.wide.su acc_hi:acc_lo, ra, rb   # signed-unsigned -> 16-bit accum

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

Signedness, accumulator, and narrowing notes:

* Signedness defaults are encoded by the mnemonic (`.s`/`.u`). If a mnemonic omits an explicit sign selector, the effective sign follows `CAP.PREC.MODE.UNS` as in the RSV base; this profile does not add alternate sign controls.
* `svmla.wide.*` accumulates into the existing 2xEW value in `rd:rd^1`; accumulators are not implicitly zeroed and must be initialized by software.
* `svnarrow.sat.*` interprets `rs2:rs1` as hi:lo of a 2xEW value and clamps to EW per the signedness variant. No rounding is performed unless a future `.rnd` variant is added.

### 6.5 Semantic Notes

* **Predication:** All XRSVS instructions consume the effective predicate defined by the XPHMG predication model (see `xphmg.md`).
* **Saturation flagging:** On any saturating operation that clamps, set `CAP.PREC.STAT.SAT_HIT`. No trap is raised. Non-clamping paths leave `SAT_HIT` unchanged.
* **Signedness and element width:** Signed/unsigned variants interpret operands and saturation bounds per `EFF_PET`/`EFF_EW` from CAP. If the mnemonic encodes signedness, CAP.UNS does not override it; otherwise signedness follows CAP.UNS (implementation choice should be documented).
* **Widening and narrowing:** Widening ops (`svmul.wide.*`, `svmla.wide.*`, `svshift.wide.s`) produce 2xEW results into `rd`/`rd^1` and require `rd` even and `SVDST.stride==1`; otherwise *illegal instruction*. Narrowing (`svnarrow.sat.*`) consumes a 2xEW pair (`rs2:rs1` as hi:lo) and saturates to EW.
* **Accumulator behavior:** `svmla.wide.*` accumulates into the existing 2xEW value in `rd:rd^1`. Accumulators are not implicitly zeroed.
* **Shift semantics:** `svshift.wide.s` performs an arithmetic shift on the 2xEW accumulator; shift amount is interpreted per the immediate rules in Section 3.
* **FP interaction:** XRSVS operates on integer lanes only; it does not define FP saturation or mixed-precision FP behavior. FP payloads are unaffected by these opcodes.
* **Illegal instruction:** Violations of pair-destination legality, reserved funct7 values, or unsupported signedness/width combinations are *illegal instruction*.

### 6.6 Notes for Implementers (Informative)

* Implementations may fuse widening multiply and accumulate paths; architectural behavior requires full 2xEW precision before narrowing.
* Toolchains should treat signedness as part of the opcode identity; do not rely on CAP.UNS to toggle signedness unless the mnemonic omits `.s`/`.u`.
* Hardware may optimize saturation detection, but must still set `SAT_HIT` consistently across masked and unmasked lanes that clamp.

---

## 7 Minimal Conformance Sets (Normative)

To keep adoption incremental, define three **levels** per profile:

Conformance levels are additive within each profile. Each level must explicitly list supported `EFF_EW` (e.g., 8/16/32). Unsupported widths are **illegal instruction**. Absence of a level implies absence of all higher levels for that profile.

* **XRSV-PRMT-M1 (Minimum):** `svzip.{lo,hi}`, `svunzip.{lo,hi}`, `svext.v`, `svshuffle.v` with patterns {000,101}. Intended as the smallest interleave/de-interleave set.
* **XRSV-PRMT-M2 (Plus):** XRSV-PRMT-M1 plus `svperm.v`, `svpermi2.v`. Adds indexed permutes across a single or dual RSV window.
* **XRSV-PRMT-F (Full):** XRSV-PRMT-M2 plus `svtbl.v`, `svtbx.v`. Adds table lookup with keep-on-OOB semantics.

* **XRSV-CEXP-M1 (Register):** `svcompress`, `svexpand`, `svpopcnt.pm`. Register-only compaction/expansion and predicate popcount.
* **XRSV-CEXP-F (Register+Memory):** XRSV-CEXP-M1 plus `svscat.cmp`, `svgath.exp` (memory forms). Enables compact scatter and gather-expand with FOF behavior.

* **XRSVS-M1 (Saturating Arithmetic):** `svadd.sat.*`, `svsub.sat.*`, `svabs.sat.s`. Basic lane-wise saturating arithmetic.
* **XRSVS-M2 (Widen/Narrow Core):** XRSVS-M1 plus `svmul.wide.*`, `svmla.wide.*`, `svnarrow.sat.*`. Adds widening multiply/MAC and saturating narrow.
* **XRSVS-F (Full):** XRSVS-M2 plus `svavg.rnd.*`, `svmin.*`, `svmax.*`, `svshift.wide.s`. Adds rounded average, min/max, and wide shift.

### 7.1 Notes for Implementers (Informative)

* Conformance is reported per profile and level. A platform may implement PRMT-F and CEXP-M1 while omitting CEXP-F or XRSVS entirely.
* Toolchains should gate instruction selection on both profile presence and level; absence must be handled via software sequences or codegen guards.
* If a level is implemented, it must support the declared `EFF_EW` set for all included instructions uniformly; partial width support within a level is not permitted.
* Additional implementation-defined levels are not permitted in v0.1; future revisions may extend these sets.

---

## 8 Programmer Hints & Mapping Guidance (Informative)

This section provides **informative mappings** between common scalar-vector idioms and the corresponding RSV-Profile instructions. These mappings are intended to help programmers and compiler authors recognize patterns; they are **non-normative** and do not imply binary or semantic compatibility with any external ISA.

Examples and guidance:

* **Permutation/shuffle idioms (XRSV-PRMT):** Use `svperm.v` for arbitrary indexed permutes within a window; `svpermi2.v` to merge from two windows under index MSB; `svshuffle.v` with predefined patterns for even/odd interleave or lane reversal; `svzip.*`/`svunzip.*` for interleave/de-interleave of two sources into a pair destination.
* **Table lookup idioms (XRSV-PRMT):** Use `svtbl.v` for in-range table lookup across the active RSV window; use `svtbx.v` for lookup-or-keep behavior on out-of-range indices.
* **Mask-based compaction/expansion (XRSV-CEXP):** Use `svcompress` to pack surviving lanes from a masked source contiguously; use `svexpand` to scatter a dense source back into masked positions; use `svpopcnt.pm` to obtain survivor counts.
* **Compact scatter/gather (XRSV-CEXP):** Use `svscat.cmp` to stream survivors contiguously to memory; use `svgath.exp` to load a dense region into masked lanes. FOF and domain/hint behavior follow `XPHMG_XMEM`.
* **Widening MAC and saturating arithmetic (XRSVS):** Use `svmul.wide.*`/`svmla.wide.*` for 2xEW products/accumulates with pair destinations; use `svnarrow.sat.*` to clamp back to EW; use `svadd.sat.*`, `svsub.sat.*`, `svabs.sat.s`, `svavg.rnd.*`, `svmin.*`, `svmax.*` for lane-wise saturated math.

Notes:

* All examples assume the active predicate defined by the XPHMG predication model; adjust predicate setup (`svsetvl`, `svon.*`, predicate ops) as needed before invoking profile instructions.
* For pair-destination forms, ensure `rd` is even and `SVDST.stride==1` to avoid *illegal instruction* traps.
* When mapping known idioms, prefer the lowest conformance level that supplies the needed instructions (see Section 7) to ease portability across implementations.
* TODO: Clarify any additional mappings with MTX/GFX/RT/ACCEL once those specifications stabilize.

---

## 9 Exceptions & Debug (Informative)

* No new sticky flags are defined by this profile set beyond `SAT_HIT` already exposed in `CAP.PREC.STAT`. Saturating operations set `SAT_HIT` when clamping; permute/table/compaction instructions do not set new flags.
* Fault-only-first behavior for memory-touching CEXP forms follows `XPHMG_XMEM`; first-fault index is recorded in `SVFAULTI` per the base RSV contract.
* Optional PMU/debug counters (non-architectural): examples include `perm_oob`, `tbl_oob`, `cmp_survivors`, `cmp_bytes`. These are implementation-defined and must not alter architectural behavior.
* Illegal-instruction traps are raised for unsupported profiles, illegal pair-dest uses, reserved encodings, or non-zero reserved immediate fields as specified in Sections 2–7.

NOTE: This section is informative. Implementations may expose additional observability via non-architectural counters or trace hooks, but architectural state and flag behavior remain governed by `XPHMG_CAP`, `XPHMG_XMEM`, and the RSV base.

---
*Licensed under CC-BY-SA 4.0 - Open Specification maintained by Pedro H. M. Garcia.*

Designed as extension for `XPHMG_RSV` and for integration with `XPHMG_CAP`, and `XPHMG_XMEM`.
