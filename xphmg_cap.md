# RISC-V Vendor Extension **XPHMG_CAP**

**Capabilities, Numeric Policy & Precision**

**Category:** Vendor Extension (`XPHMG_CAP`)
**Version:** 0.1.1
**Author:** Pedro H. M. Garcia
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview

`XPHMG_CAP` is the single source of architectural truth for the XPHMG family. It aggregates capability descriptors and the runtime numeric/memory policy visible to all XPHMG execution domains.

### 1.1 Scope and role

**Normative requirements:**
1. `XPHMG_CAP` is the sole architectural authority for capability discovery and for numeric/memory policy state consumed by all XPHMG domains (RSV, MTX, GFX, RT, XMEM, ACCEL).
2. Implementations MUST expose capability discovery fields as read-only, non-binding descriptors of implementation support.
3. Implementations MUST store architectural numeric and memory policy in `XPHMG_CAP` state; all XPHMG domains MUST execute using only this state.
4. No XPHMG extension MAY redefine, override, or bypass semantics defined by `XPHMG_CAP`.
5. At each instruction boundary, every XPHMG engine MUST snapshot the effective `XPHMG_CAP` state and execute deterministically under that configuration until completion.

**Informative:** `XPHMG_CAP` owns numeric behavior (precision, element width, accumulator width, rounding, saturation, quantization, NaN policy, exception handling) and memory behavior (memory classes, coherence domains, streaming, gather/scatter). Centralizing these controls avoids divergence between domains and toolchains.

### 1.2 Conceptual model

* Architectural state: `CAP.PREC.*` and `CAP.XMEM.*` are mandatory, latched at instruction boundaries, and are the only architectural sources of numeric and memory defaults for all XPHMG domains. Per-instruction overrides (if present in a domain ISA) affect only that instruction and do not modify CAP state.
* Advisory state: `CAP.FEAT*`, `CAP.TIER.*`, `CAP.HINT.*`, `CAP.PREC.{ALU,TEX,IMG,RT}` describe implementation capabilities and tuning hints. They are descriptive and non-binding unless another section states otherwise.
* Profiles: `CAP.PROF.*` (if `HAS_PROFILES=1`) select among advisory variants only when `RUNTIME_MUTABLE=1`; they do not change architectural numeric or memory semantics.
* Determinism: APPLY gates (PREC) and instruction-boundary latching (XMEM/SVMEM) provide deterministic observation and replay across all domains.

### 1.3 Architectural guarantees & invariants

* Single source: Numeric precision/exception policy and memory/gather-scatter defaults are architecturally defined only by `CAP.PREC.*` and `CAP.XMEM.*`.
* Non-overriding: Downstream XPHMG extensions (RSV, MTX, RT, GFX, XMEM) must reference CAP state rather than redefining it; engine-local mirrors are micro-architectural only.
* Snapshot execution: Engines execute using the CAP state visible at decode/issue; mid-instruction CAP writes MUST NOT affect in-flight operations.
* Advisory stability: Advisory CSRs are read-only; they are stable after reset unless `CAP.FLAGS.RUNTIME_MUTABLE=1` explicitly permits variation.
* Reserved/unsupported: Writes to reserved or unsupported fields MUST be ignored and read as zero; unsupported numeric formats are flagged via `CAP.PREC.STAT.UNSUP_FMT`.

### 1.4 Interaction with other XPHMG extensions (informative)

* RSV (vector) uses CAP for default numeric policy (precision, rounding, NaN/saturation, masked-lane `ZMODE`) and for SVMEM gather/scatter defaults; per-op prefixes may override for a single instruction without mutating CAP.
* XMEM consumes CAP memory defaults (`CAP.XMEM.*`) as the architectural programming model; engine-local controls, if any, are mirrors and MUST NOT change semantics.
* GFX/MTX/RT domains consume CAP numeric and memory defaults; format capability descriptors (`CAP.PREC.{ALU,TEX,IMG,RT}`) inform toolchain selection of supported types.
* Profiles (if present) are discoverable via `CAP.PROF.*` and may co-vary advisory tiers/hints; they MUST NOT alter CAP-defined correctness semantics.

### 1.5 Undefined / reserved behavior

* Behavior is UNDEFINED if software depends on staged-but-not-applied PREC state or assumes advisory bits imply correctness guarantees.
* Profile encodings outside `CAP.PROF.SUP` are reserved; reading such values yields implementation-defined behavior that MUST NOT be relied upon.

### 1.6 Notes for implementers (informative)

* Initialize CAP state before dispatching work to any XPHMG domain; stage PREC writes then set APPLY to avoid transient observability.
* Debug/replay flows should capture CAP state (including latched IE_MASK) to reproduce execution deterministically across domains.
* Toolchains should treat advisory CSRs as hints only; correctness must derive from explicit programming of `CAP.PREC.*` and `CAP.XMEM.*`.

---

## 2. Architectural Model

### 2.1 Numeric Model Overview

`CAP.PREC.*` defines the architectural numeric policy consumed by RSV, MTX, GFX, RT, and XMEM. Per-instruction overrides (e.g., RSV prefixes) supersede the defaults for that instruction only and do not modify `CAP.PREC.*`.

**Conceptual model (informative):**
* A single effective numeric state (`EFF`) is derived from `CAP.PREC.MODE` and optionally merged with `CAP.PREC.ALT` when ALT is enabled and supported. `EFF` is captured at instruction boundary and remains constant for the lifetime of the instruction in all domains.
* `CAP.PREC.EXC.*` supplies the trap/sticky policy; SAE defaults in CAP may be further constrained by per-instruction overrides when provided by a domain ISA.
* Quantization controls (`Q`, `ZP_*`, `PACK`) and NaN/FTZ/SAE policies are part of the architectural numeric state and apply uniformly across domains that implement the corresponding behaviors.

**Architectural guarantees & invariants:**
* All domains MUST consume the same `EFF` snapshot at decode/issue; mid-instruction CAP writes MUST NOT affect execution.
* Unsupported ALT formats do not alter `EFF`; they are reported via `CAP.PREC.STAT.UNSUP_FMT` while base MODE remains effective.
* Per-instruction overrides (if present) are purely one-shot and MUST NOT modify CAP architectural state.

**Interaction with other extensions (informative):**
* RSV vector masked writes follow `EFF_ZMODE` unless an RSV one-shot prefix overrides it for a single instruction.
* MTX/RT/GFX datapaths consume the same `EFF` for rounding, NaN, saturation, and accumulation width; toolchains should program CAP once per phase and avoid per-engine divergence.
* XMEM units that perform numeric conversion (e.g., compression/expansion) use `EFF` for saturation/rounding unless an XMEM descriptor explicitly overrides.

**Notes for implementers (informative):**
* Stage writes to MODE/ALT then set APPLY to avoid partial visibility; capture `CAP.PREC.STAT` after APPLY to confirm the effective state latched.
* Debug/replay flows should record `CAP.PREC.STAT` (including `IE_MASK`) to reproduce numeric behavior across domains.

TODO: Clarify interaction with domain-specific SAE/ZMODE overrides in RSV/GFX/RT specs where present.

### 2.2 Memory and Tier Model (Global Policy)

`XPHMG` defines a unified memory and dataflow model for all accelerators (RSV, MTX, GFX, RT, ACCEL). Inter-engine communication MUST use the tiered memory hierarchy managed by `CAP.XMEM.*` and `XPHMG_MEMCTL`; direct cross-pipeline register forwarding is non-architectural.

**Normative requirements:**
1. Inter-engine dataflow MUST use architecturally visible tiers; private register files are non-architectural.
2. Tier-0 is the sole architecturally visible tier that enables cross-engine handoff without data movement; changing logical ownership (`dom`) of a Tier-0 region constitutes a handoff.
3. Tiers beyond Tier-0 MUST follow the coherence, class, and streaming policies defined by CAP and `XPHMG_MEMCTL`.

#### 2.2.1 Tier Model (Architectural)

Implementations MUST expose at least the following conceptual tiers:

| Tier | Description |
|------|-------------|
| **Tier-0 (T0)** | Small, low-latency LDS/XMEM region treated as quasi-register storage for hot cross-engine data. |
| **Tier-1 (T1)** | LDS/XMEM scratchpad, medium latency, per-stage working sets. |
| **Tier-2 (T2)** | Coherent caches / shared-memory hierarchy. |
| **Tier-3 (T3)** | DRAM or external memory. |

Tier-0 is the unique architecturally visible level where cross-engine handoff occurs without copying. Private engine register files are out of scope.

#### 2.2.2 Global Tier Capabilities (`CAP.TIER`)

`CAP.TIER` (RO) SHOULD describe: supported tier count (minimum T0 and T1), maximum Tier-0 capacity, whether Tier-0 is inside LDS, whether Tier-0 is visible to scalar/RSV, and whether Tier-0 supports cross-engine shared access (`T0_SHARED=1`). Undefined bits MUST read zero.

#### 2.2.3 Global Dataflow Policy

When multiple engines participate in a pipeline (e.g., MTX->RT->GFX), data exchanges MUST use Tier-0 for hot/low-latency paths and Tier-1 as fallback when Tier-0 capacity is insufficient. Engine-to-engine transfers MUST NOT imply physical copies; handoff is defined by ownership of the tiered region.

#### 2.2.4 Tier-1 Working Set Policy

Tier-1 (T1) is the primary working-set level, implemented in LDS/XMEM or equivalent low-latency local memory and visible to all engines.

**Normative requirements:**
1. Engines MUST treat Tier-1 as the default target for local working sets that do not require Tier-0 latency and do not require global coherence (Tier-2+).
2. Cross-engine communication that does not require Tier-0 latency MAY use Tier-1 following XMEM ordering and synchronization.

Tier-1 does not define data format; it defines placement/latency. Layout/tiling is governed by XDL and `XPHMG_MEMCTL`.

##### 2.2.4.1 Tier-1 Capability Hints

`CAP.TIER`/`CAP.HINT` SHOULD expose Tier-1-related descriptive limits (e.g., typical LDS budget per WG, preferred bank/granularity). Toolchains may use these as heuristics for tile sizing and LDS budgeting; they do not change correctness.

TODO-ELABORATE: Expand Tier-1 capability hints and connect to CAP.TIER/CAP.TILE.

**Architectural guarantees & invariants:**
* CAP.XMEM state (classes, coherence, streaming, descriptors) is the single architectural source of memory defaults for all domains; domain-local mirrors are micro-architectural and MUST NOT change semantics.
* SVMEM defaults (`CAP.XMEM.SVM*`) apply to indexed gather/scatter across RSV/GFX/MTX/RT; predication is sourced from PMASK (effective predication) and masked-lane register writeback follows `CAP.PREC.MODE.ZMODE` while masked lanes MUST NOT issue memory accesses.
* Tier ownership changes (handoff) MUST obey CAP.XMEM ordering and coherence rules; Tier-0 handoff is the only architectural cross-engine path without copying.

**Interaction with other extensions (informative):**
* XMEM describes implementation-specific mechanisms but consumes CAP.XMEM defaults as the architectural contract.
* RSV/GFX/MTX/RT indexed memory ops default to CAP.XMEM.SVM* unless a per-descriptor or per-op override is defined in the respective ISA.
* ACCEL (if present) must also consume CAP.XMEM defaults for coherence/class/streaming to interoperate with other domains.

**Notes for implementers (informative):**
* Program CAP.XMEM.* before dispatch; changes take effect at the next instruction boundary without APPLY.
* If a feature bit in `CAP.XMEM.CAP` is 0, writes to dependent fields are ignored; software should probe capabilities before programming optional features.

### 2.3 Predication Model (PMASK-owned state, CAP-owned policy)

`XPHMG_CAP` does not define predicate state. Predicate state is defined by `XPHMG_PMASK` and consumed by XPHMG domains.
CAP defines the **policy invariants** that bind predication across domains (writeback behavior and memory side-effects).

**Normative requirements:**
1. **Architectural source:** Any XPHMG domain that implements predication MUST source predication from `XPHMG_PMASK` (effective predication), not from domain-local mask state.
2. **Effective predication:** For each operation, the effective predicate is the PMASK bank selected by the operation’s `pbank` (explicitly encoded by the consuming extension or implied by its enable/prefix rules). If an operation does not select a bank, `pbank=0` is implied.
3. **bank0 semantics:** If `XPHMG_PMASK` is present, `pbank=0` MUST behave as a virtual ALL-ONES predicate (reads as all-ones for the effective predicate width; writes ignored).
4. **Writeback policy:** For any architecturally visible register/vector destination element whose predicate is false, the destination MUST follow `CAP.PREC.MODE.ZMODE`:
   - `ZMODE=0` (**merge**): preserve the prior destination element.
   - `ZMODE=1` (**zero**): write architectural zero to the destination element.
5. **Memory side-effects:** Predicated memory operations MUST NOT issue memory accesses for predicate-false elements/lanes. This rule applies uniformly to scalar predication, lane-based predication, and indexed gather/scatter (SVMEM).

NOTE: CAP defines the policy above; the selecting/encoding of `pbank` and any predicate-producing instructions are defined by the consuming extensions (e.g., RSV/RT/GFX) and by `XPHMG_PMASK` itself.


### 2.4 Cross-Domain Execution, Masks, Exceptions

NOTE: Normative cross-domain rules are consolidated in Section 5.

**Conceptual overview (informative):**
* Cross-domain determinism relies on a common CAP snapshot: numeric (`EFF`) and memory (`CAP.XMEM.*`, SVMEM) are latched per instruction boundary in every engine.
* Masked execution semantics (merge vs zero) are governed by `CAP.PREC.MODE.ZMODE` for any domain that implements predication; domains without masking are unaffected.
* FP exception observation/trapping is unified via `CAP.PREC.EXC.*`; SAE defaults are set in CAP and may be overridden per-instruction when supported by a domain.

**Interaction with other extensions (informative):**
* RSV one-shot prefixes (if implemented) can override RC/SAE/ZMODE for one instruction; absent such overrides, all domains use CAP defaults.
* Memory ordering, coherence, and streaming defaults come from CAP.XMEM and are consumed by all domains; domain-specific descriptors may narrow but not expand beyond CAP-defined semantics.

**Notes for implementers (informative):**
* When mixing domains in a pipeline, ensure CAP state is programmed and applied before dispatch; avoid relying on engine-local defaults.
* Use `CAP.PREC.STAT` and `CAP.XMEM.*` reads to confirm the effective state before issuing mixed-domain workloads.

---

## 3. CSR Address Map

`XPHMG_CAP` CSRs occupy the vendor CSR window `0x7C0-0x7FF`. This section enumerates the suggested partitioning of that window across XPHMG sub-extensions.

**Normative requirements:**
1. If an implementation adopts the suggested vendor CSR map, each address range in sections 3.1-3.x MUST map to the block described in the corresponding subsection.
2. CSRs listed in this map that are not implemented MUST return zero on reads and MUST ignore writes unless a subsection specifies another behavior.
3. Implementations claiming XPHMG conformance MUST NOT alias the `0x7C0-0x7FF` ranges with unrelated vendor CSRs.

**Conceptual model (informative):**
* The vendor window is partitioned into non-overlapping blocks owned by CAP or related XPHMG extensions; each block is defined normatively in the section indicated by the table below.
* Addresses not implemented within a block read as zero and ignore writes; no aliasing with other vendor CSRs is permitted.
* The CAP-defined map is advisory for physical placement but normative for discoverability: software probing uses these addresses to locate CAP fields without implementation-specific indirection.

**Architectural guarantees & invariants:**
* Address ranges are inclusive and non-overlapping; no CSR may span multiple ranges.
* `0x7F8-0x7FF` is reserved for `XPHMG_RSV` and is defined in that specification; CAP does not assign semantics inside that sub-window.
* Detailed field definitions reside in Section 4; this section establishes only the address map.

**Notes for implementers (informative):**
* If a block is unimplemented, all addresses in its range MUST read zero and ignore writes; do not repurpose unimplemented CAP ranges for other functions.
* Toolchains rely on this map for capability discovery; deviations from the map require explicit documentation and will break software assumptions.

### 3.1 High-level XPHMG CSR layout

To avoid collisions between XPHMG sub-extensions, the vendor CSR window `0x7C0-0x7FF` is partitioned as follows. All addresses are inclusive and all CSRs are XLEN-wide.

**Normative requirements:**
1. The address ranges in Table 3.1 MUST be treated as inclusive and non-overlapping.
2. Implementations MUST expose each CSR they implement at the exact address shown in Table 3.1.
3. CSRs within these ranges that are not implemented MUST return zero on reads and MUST ignore writes unless another subsection specifies different behavior.

| Range         | Owner / Block         | Contents                                                    |
| ------------- | --------------------- | ----------------------------------------------------------- |
| `0x7C0-0x7CF` | `XPHMG_CAP` (core)    | ID, version, global flags, profiles, features, tiers, hints |
| `0x7D0-0x7D7` | `XPHMG_CAP.PREC`      | Numeric/precision policy & FP exception CSRs                |
| `0x7D8-0x7DF` | `XPHMG_CAP.PREC.CAP`  | Precision/format capability descriptors (see 4.5)           |
| `0x7E0-0x7EF` | `XPHMG_CAP.XMEM`      | Memory / LDS / streaming / descriptor policy                |
| `0x7F0-0x7F7` | `XPHMG_CAP.XMEM.SVM`  | Indexed gather/scatter (SVMEM) configuration                |
| `0x7F8-0x7FF` | `XPHMG_RSV`           | Scalar-Vectors control CSRs (see XPHMG_RSV)                 |

**Informative:** This document normatively defines the `XPHMG_CAP`-owned ranges; `XPHMG_RSV` defines the semantics of its own CSRs within the reserved `0x7F8-0x7FF` sub-window.

---

## 4. CSR Definitions

All bitfields in this section follow these general rules unless a subsection states otherwise:

**Normative rules:**
1. RO bits are constant for a given implementation and MUST ignore writes.
2. Bits not defined in a table MUST read as zero and MUST ignore writes.
3. RO/W1C bits clear only those positions written as 1 and ignore zeros.
4. `APPLY*` gates (`APPLY0`, `APPLY1`) are write-one-to-apply. Writes without an APPLY bit MAY be staged internally but MUST NOT change architectural state until an APPLY write is performed for the same group.
5. `CAP.PREC.*` state is latched at the instruction boundary by all XPHMG domains; in-flight instructions execute with the state captured at decode.

**Conceptual model (informative):**
* Identification and advisory CSRs are descriptive and read-only; they do not alter execution semantics.
* Numeric policy (`CAP.PREC.*`) and memory policy (`CAP.XMEM.*`, including SVMEM) are architectural; they are the single source of truth for all XPHMG domains.
* Profiles (`CAP.PROF.*`) are descriptive selectors gated by `HAS_PROFILES`; they never mutate architectural numeric or memory state.

**Notes for implementers (informative):**
* Program architectural RW CSRs (PREC/XMEM/SVMEM) before dispatching work; PREC requires APPLY, XMEM/SVMEM latch at instruction boundary.
* Probe capability CSRs (`CAP.FEAT*`, `CAP.PREC.*` capability blocks, `CAP.TIER.*`, `CAP.HINT.*`) before programming optional features; unimplemented bits read zero and ignore writes where applicable.

### 4.1 Identification & Version

#### 4.1.1 Core Identification & Versioning (RO)

These CSRs identify the XPHMG_CAP provider and advertise versioning and profile state.

**Normative requirements:**
1. All CSRs in Table 4.1.1 MUST be read-only.
2. `CAP.ID` MUST encode the vendor/architecture identifier assigned to the implementation (e.g., 0x50484D47 for "PHMG").
3. `CAP.VERS` MUST encode the layout version as Major:8, Minor:8, Patch:8, Reserved:8.
4. `CAP.PROF.CUR` and `CAP.PROF.SUP` MUST be implemented only if the profile mechanism is present (HAS_PROFILES=1); otherwise they MUST return zero.
5. `CAP.FLAGS` MUST report the global flag bits defined in section 4.1.2.

| CSR            | Addr  | Access | Description                                                  |
| -------------- | ----- | ------ | ------------------------------------------------------------ |
| `CAP.ID`       | 0x7C0 | RO     | Vendor/arch ID (e.g., 0x50484D47 "PHMG").                    |
| `CAP.VERS`     | 0x7C1 | RO     | Layout version (Major:8, Minor:8, Patch:8, Rsvd:8).          |
| `CAP.FLAGS`    | 0x7C2 | RO     | Global flags (see section 4.1.2).                            |
| `CAP.PROF.CUR` | 0x7C3 | RO     | Current runtime profile (valid if HAS_PROFILES=1).           |
| `CAP.PROF.SUP` | 0x7C4 | RO     | Supported profiles bitmask (valid if HAS_PROFILES=1).        |

**Informative:** If the profile mechanism is unimplemented, `CAP.PROF.*` may instead be described in section 4.3, and addresses 0x7C3/0x7C4 will read as zero.

#### 4.1.2 `CAP.FLAGS` (RO)

`CAP.FLAGS` is a read-only discovery CSR. Bits are architectural and stable after reset.

* `RUNTIME_MUTABLE` (bit 0): Advisory RO descriptors (`CAP.FEAT*`, `CAP.TIER.*`, `CAP.HINT.*`) may legally vary with power/thermal or firmware policy. Architectural state (`CAP.PREC.*`, `CAP.XMEM.*`) MUST remain stable unless software updates it explicitly.
* `HAS_PROFILES` (bit 1): If 1, `CAP.PROF.*` in section 4.3 are implemented and may describe multiple runtime profiles. If 0, `CAP.PROF.*` MUST read zero; profile-driven variation of advisory fields MUST NOT be exposed unless `RUNTIME_MUTABLE=1`.
* Bits XLEN-1:2 are reserved, read zero, and ignore writes.

### 4.2 Discovery & Advisory Capabilities

#### 4.2.1 Feature Bits (RO, non-binding)

These CSRs expose non-binding feature discovery bits for software capability probing.

**Normative requirements:**
1. All CSRs in Table 4.2.1 MUST be read-only.
2. Bits not defined in Table 4.2.1 MUST read as zero and MUST be ignored by software.
3. Feature bits are non-binding: software MUST treat a set bit as *"supported by the implementation"* and a clear bit as *"unsupported or not exposed"*.

| CSR             | Addr  | Access | Description                                                          |
| --------------- | ----- | ------ | -------------------------------------------------------------------- |
| `CAP.FEAT0`     | 0x7C5 | RO     | General features (bindless, OOO mem, RT, etc.).                      |
| `CAP.FEAT1`     | 0x7C6 | RO     | Sync/barrier features, PMU presence.                                 |
| `CAP.PMASK.CAP` | 0x7C7 | RO     | Predicate-mask discovery (presence, bank count, bank0 ALL-ONES rule).|

**Informative:** Feature bits group optional capabilities; they do not alter architectural behavior without explicit enablement elsewhere.

`CAP.FEAT0` advertises optional capabilities for software tuning. Bits are descriptive, non-binding, and stable after reset unless `RUNTIME_MUTABLE=1` explicitly permits variation. Clearing a bit does not forbid equivalent functionality provided through another mechanism.

Suggested mapping (non-exhaustive):

* `BINDLESS`: Bindless resource access in GFX/RT (see `XPHMG_GFX`).
* `OOO_MEM`: Out-of-order memory scheduling permitted by XMEM; ordering/visibility still follow fences/scopes.
* `DYN_REG_ALLOC`: Dynamic VGPR/SGPR allocation in RSV (see `XPHMG_RSV`).
* `RT_PRESENT`: Ray-tracing pipeline present (see `XPHMG_RT`).
* `LDS_ATOMICS`: LDS atomics supported in the XMEM/LDS path.
* `ANISO`: Anisotropic filtering supported by the texture pipeline.
* `IMG_ATOMICS`: Typed image/buffer atomics supported (see `XPHMG_GFX_IMG`).
* `SLC_PRESENT`: System-level cache coherence hooks present.
* `EXT_BARRIERS`: Extended barrier/synchronization primitives implemented (XMEM / RSV sync model).
* `PMU_PRESENT`: XPHMG-specific performance counters present.

Undefined bits read zero and MUST be ignored by software.

##### 4.2.1.1 Predicate-mask discovery (`CAP.PMASK.CAP`) (RO, non-binding)

`CAP.PMASK.CAP` provides capability discovery for the **architectural predication substrate** defined by `XPHMG_PMASK`.
This CSR is descriptive only; it does not enable predication by itself.

**Normative requirements:**
1. If `PMASK_PRESENT=0`, the implementation does not expose `XPHMG_PMASK` architectural state. Any predication-dependent behavior in other extensions is unavailable unless that extension defines a software fallback; CAP does not define such a fallback.
2. If `PMASK_PRESENT=1`, the implementation MUST provide banked predicate state as defined by `XPHMG_PMASK`, including the bank0 ALL-ONES rule.
3. `PMASK_BANKS` reports the number of architecturally visible predicate banks (including bank0). Baseline XPHMG expects 4 banks.

**Bit layout (v0.1):**
- bit[0]  `PMASK_PRESENT` : 1 if `XPHMG_PMASK` predicate state is implemented.
- bits[3:1] `PMASK_BANKS_M1` : bank count minus one (`banks = PMASK_BANKS_M1 + 1`).
- bit[4]  `BANK0_ALLONES` : 1 if `pbank=0` is a virtual ALL-ONES bank (MUST be 1 when `PMASK_PRESENT=1`).
- bits[XLEN-1:5] reserved, read as zero.

TODO-ELABORATE: Define FEAT0 bit positions and add missing FEAT1 table.

#### 4.2.2 Tiers & Hints (RO, non-binding)

These CSRs provide non-binding capacity tiers and implementation hints for performance-sensitive software.

**Normative requirements:**
1. All CSRs in Table 4.2.2 MUST be read-only.
2. Tier and hint encodings are descriptive only; software MUST treat them as guidance, not as guarantees.
3. Bits not defined in Table 4.2.2 MUST read as zero and MUST be ignored by software.

| CSR              | Addr  | Access | Description                                   |
| ---------------- | ----- | ------ | --------------------------------------------- |
| `CAP.TIER.REG`   | 0x7C8 | RO     | VGPR/SGPR tiers (S/M/L/XL).                   |
| `CAP.TIER.WAVE`  | 0x7C9 | RO     | Typical waves per CU ({8,16,24,32}).          |
| `CAP.TIER.LDS`   | 0x7CA | RO     | Typical LDS budget per WG (tiered).           |
| `CAP.TIER.SAMP`  | 0x7CB | RO     | Sampler queue depth tier.                     |
| `CAP.TIER.RT`    | 0x7CC | RO     | RT pipeline capacity tier.                    |
| `CAP.HINT.ALIGN` | 0x7CD | RO     | Preferred alignments (bitmask: 16/32/64/128). |
| `CAP.HINT.BVHGR` | 0x7CE | RO     | Preferred BVH prefetch granularity.           |
| `CAP.HINT.LDSBK` | 0x7CF | RO     | LDS bank granularity hint.                    |

**Informative:** Tiers guide compiler and runtime heuristics (e.g., register allocation, work-group sizing), while hints inform layout and prefetch strategies. They do not change architectural guarantees or correctness.

`CAP.TIER.*` and `CAP.HINT.*` are read-only and descriptive. Values MAY vary only if `CAP.FLAGS.RUNTIME_MUTABLE=1`; otherwise they are stable after reset unless modified by a future revision.

* `CAP.TIER.REG`: `{VGPR_TIER, SGPR_TIER}` (S/M/L/XL).
* `CAP.TIER.WAVE`: waves per CU (encoded ranges).
* `CAP.TIER.LDS`, `CAP.TIER.SAMP`, `CAP.TIER.RT`: relative tiers.
* `CAP.HINT.ALIGN`: bitmask of preferred alignments (16/32/64/128).
* `CAP.HINT.BVHGR`: preferred `RAY.NODE.PREFETCH` granularity (e.g., encoded {64,128,256}).
* `CAP.HINT.LDSBK`: LDS bank hint.

TODO-ELABORATE: Align tier and hint fields with conceptual Tier model and CAP.TIER/CAP.TILE.

### 4.3 Profiles (RO)

Runtime profiles are an optional, descriptive mechanism. They do not introduce new architectural state; they select among pre-described advisory variants when permitted by `CAP.FLAGS.RUNTIME_MUTABLE`.

#### 4.3.1 Presence and discovery (Normative)

* Profiles are present only if `CAP.FLAGS.HAS_PROFILES=1`.
* When present, `CAP.PROF.CUR` exposes the active profile identifier; `CAP.PROF.SUP` exposes the bitmask of supported profiles. Both CSRs are read-only.
* If `HAS_PROFILES=0`, both CSRs MUST read zero and MUST ignore writes.

| CSR            | Addr  | Access | Description                       |
| -------------- | ----- | ------ | --------------------------------- |
| `CAP.PROF.CUR` | 0x7C3 | RO     | Current profile (e.g., P0/P1/P2). |
| `CAP.PROF.SUP` | 0x7C4 | RO     | Bitmask of supported profiles.    |

#### 4.3.2 Scope and effect (Normative)

* Profiles are descriptive only. They MUST NOT change architectural numeric or memory semantics defined by `CAP.PREC.*` or `CAP.XMEM.*`.
* Profiles MAY select among advisory tiers/hints (`CAP.TIER.*`, `CAP.HINT.*`, potentially `CAP.FEAT*`) only if `CAP.FLAGS.RUNTIME_MUTABLE=1`. If `RUNTIME_MUTABLE=0`, advisory CSRs are stable after reset regardless of profile selection.
* Switching profiles MUST NOT cause mid-instruction changes; profile observability is latched at instruction boundaries like other CAP state.

#### 4.3.3 Selection and transition (Normative)

* The mechanism to change profiles (e.g., firmware, privileged write, or job submission metadata) is implementation-defined and outside this specification; however, any change MUST update `CAP.PROF.CUR` before it becomes architecturally visible to instructions.
* Profile changes MUST NOT implicitly modify `CAP.PREC.*` or `CAP.XMEM.*`; if software needs different numeric or memory policy, it MUST program those CSRs explicitly.
* Forward progress: entering or exiting a profile MUST NOT deadlock engines; implementations may stall until profile state is coherently visible.

#### 4.3.4 Undefined / reserved behavior (Normative)

* Behavior is UNDEFINED if software assumes a profile changes correctness or exception semantics beyond advisory fields.
* Writes to `CAP.PROF.*` are ignored; any attempt to write MUST NOT trap.
* Profile encodings outside `CAP.PROF.SUP` are reserved; reading them in `CAP.PROF.CUR` implies implementation-defined behavior and MUST NOT be relied upon.

**Informative:** Profiles provide a descriptive hook for power/thermal/performance-tuned advisory settings. Toolchains and runtimes should treat profiles as hints; correctness must derive solely from explicit `CAP.PREC.*` and `CAP.XMEM.*` programming.

### 4.4 Numeric Policy (architectural state)

#### 4.4.1 Precision & Numeric Policy CSRs

These CSRs hold mandatory numeric policy state that governs all XPHMG domains.

**Normative requirements:**
1. Access types in Table 4.4.1 MUST be enforced: RW fields are writable; RO fields are read-only; RO/W1C fields clear only the bits written as 1 and ignore 0 writes.
2. Bits not defined in Table 4.4.1 MUST read as zero and MUST be ignored on write unless the access type is RO, in which case writes follow standard CSR semantics.
3. `CAP.PREC.STAT` MUST report the effective numeric state, including any stickies accumulated from prior operations.
4. `CAP.PREC.EXC.EN` bits 4:0 MUST map to NV, DZ, OF, UF, NX in that order.
5. `CAP.PREC.EXC.ST` MUST expose sticky flags NV, DZ, OF, UF, NX plus QNAN_SEEN and SNAN_SEEN; writing a 1 to any implemented bit MUST clear only that bit.
6. `CAP.PREC.RSV0` MUST read as zero and MUST ignore writes.

| CSR               | Addr  | Access | Description                                                           |
| ----------------- | ----- | ------ | --------------------------------------------------------------------- |
| `CAP.PREC.MODE`   | 0x7D0 | RW     | Base precision & numeric policy.                                      |
| `CAP.PREC.ALT`    | 0x7D1 | RW     | Alternate/extended format control.                                    |
| `CAP.PREC.STAT`   | 0x7D2 | RO     | Effective state + stickies.                                           |
| `CAP.PREC.EXC.EN` | 0x7D3 | RW     | FP exception enables: NV,DZ,OF,UF,NX (bits 4:0).                      |
| `CAP.PREC.EXC.ST` | 0x7D4 | RO/W1C | Sticky flags NV,DZ,OF,UF,NX + QNAN_SEEN,SNAN_SEEN (write 1 to clear). |
| `CAP.PREC.RSV0`   | 0x7D5 | RO     | Reserved for future precision controls (reads zero).                  |

**Informative:** This block centralizes numeric precision, exception enables, and sticky reporting so that all domains observe consistent policy and error handling.

#### 4.4.2 Numeric Policy Field Definitions

##### 4.4.2.1 `CAP.PREC.MODE` - Base Precision Control (RW)

**Purpose:** default numeric type/behavior for GFX/ALU, MTX, RT, XMEM.

| Bits   | Name           | Description                                                                  |
| ------ | -------------- | ---------------------------------------------------------------------------- |
| XLEN-1 | **APPLY0**     | Write-one-to-apply. Latches fields atomically; auto-clears.                  |
| 30     | **SAT**        | Saturate on conversion overflow (0 = trap/default, 1 = saturate on writeback). |
| 29     | **SAE_DEF**    | Suppress-all-exceptions default for FP ops (0 = use enables/traps, 1 = stickies only). |
| 28:27  | **FP_RMODE**   | Default rounding mode (RNE/RZ/RDN/RUP/RTZ/RSV).                              |
| 26:24  | **ACCW**       | Accumulator width (00=FP32, 01=FP16, 10=INT32, 11=INT16).                    |
| 23:21  | **PET**        | Primary element type selector (FP16/FP32/BF16/INT8/INT16/INT32/RSV).         |
| 20:19  | **EW**         | Element width (00=8, 01=16, 10=32, 11=64).                                   |
| 18:17  | **PACK**       | Packing factor (0=scalar, 1=2-way, 2=4-way).                                 |
| 16     | **SAT_MODE**   | Saturation mode selector (0 = clamp, 1 = alt).                               |
| 15:14  | **ZP_EN**      | Zero-point enable for quantized modes (00=off, 01=per-tensor, 10=per-channel). |
| 13:6   | **ZP**         | Zero-point value or base.                                                    |
| 5      | **Q**          | Quantization enable.                                                         |
| 4      | **UNS**        | Unsigned quantization (if Q=1).                                              |
| 3      | **ZMODE**      | Masked-lane merge/zero policy (0=merge,1=zero).                              |
| 2      | **SAE**        | One-shot suppress-all-exceptions (sticky only).                              |
| 1:0    | **RSV**        | Reserved.                                                                    |

##### 4.4.2.2 `CAP.PREC.ALT` - Alternate/Extended Mode (RW)

| Bits   | Name            | Description                                                                 |
| ------ | --------------- | --------------------------------------------------------------------------- |
| XLEN-1 | **APPLY1**      | Write-one-to-apply ALT. Auto-clears.                                       |
| 30     | **ALT_EN**      | Enable ALT mode.                                                           |
| 29:26  | **ALT_FMT**     | ALT format selector (see ALT mapping).                                     |
| 25:24  | **ALT_ACCW**    | ALT accumulator width override (0 = inherit MODE.ACCW).                    |
| 23     | **MIXED**       | Mixed-mode enable (e.g., FP8->FP16 acc).                                   |
| 22     | **SAT**         | ALT-specific saturation (priority over MODE.SAT).                          |
| 21:14  | **ZP**          | ALT zero-point.                                                            |
| 13:6   | **ZP_OFFS**     | Zero-point offset.                                                         |
| 5:3    | **PACK**        | Minimum packing factor required by ALT.                                    |
| 2:0    | **VER**         | Version/catalog index for the ALT format table.                            |

**ALT Mapping (normative)**

| ALT_FMT   | Effective PET | EW | Notes                                |
| --------- | ------------- | -- | ------------------------------------ |
| BF16_LITE | BF16          | 16 | Reduced exponent/mantissa precision. |
| FP8_E4M3  | FP8           | 8  | DL float.                            |
| FP8_E5M2  | FP8           | 8  | Transformer float.                   |
| INT4      | INT4          | 8  | Two INT4 values per byte; forces `PACK = 1`. |
| INT2      | INT2          | 8  | Four INT2 values per byte; forces `PACK = 2`. |

##### 4.4.2.3 Composition Rules (Effective State `EFF`)

* If `ALT_EN=0`: `EFF = CAP.PREC.MODE`.
* If `ALT_EN=1`: merge field-wise:
  * `PET, EW` per ALT mapping table.
  * `ACCW` uses `ALT_ACCW` when non-zero; otherwise inherits `MODE.ACCW`.
  * `SAT` uses `ALT_SAT` when 1; otherwise inherits `MODE.SAT`.
  * `Q, UNS, FP_RMODE, DENORM_FTZ` inherit from `MODE`.
  * `PACK` inherits from `MODE` unless ALT requires a minimum (INT4/INT2 force PACK >= requirement).
  * `ZP_EN/ZP/SCALE_SH` apply only if `MODE.Q=1`.
  * `EFF_ZMODE`, `EFF_SAE`, `EFF_NAN_POL` mirror the defaults in use.
  * `IE_MASK` is the latched view of `EXC.EN` after SAE is applied.
* Unsupported `ALT_FMT` values MUST leave `EFF` equivalent to `CAP.PREC.MODE`, set `CAP.PREC.STAT.UNSUP_FMT=1`, and MAY clear `ALT_EN` in `CAP.PREC.STAT`.
* Implementations MUST recompute `EFF` whenever `APPLY0=1` or `APPLY1=1` is written. Writes without APPLY MAY be staged but are not visible architecturally.
* After `APPLY0=1` or `APPLY1=1`, implementations MUST latch the effective `IE_MASK` (`CAP.PREC.EXC.EN & ~SAE_effective`) into `CAP.PREC.STAT.IE_MASK` for debug and replay determinism.

##### 4.4.2.4 `CAP.PREC.STAT` - Effective State & Sticky Flags (RO)

|  Bits | Name               | Description                                                   |
| ----- | ------------------ | ------------------------------------------------------------- |
| 31:28 | **EFF_PET**        | Effective primary element type.                               |
| 27:26 | **EFF_EW**         | Effective element width.                                      |
| 25:24 | **EFF_ACCW**       | Effective accumulator width.                                  |
|    23 | **EFF_ALT_EN**     | Alternate mode active.                                        |
|    22 | **EFF_SAT**        | Effective saturation.                                         |
| 21:19 | **EFF_FP_RMODE**   | Current rounding mode.                                        |
|    18 | **EFF_Q**          | Quantized path enabled.                                       |
| 17:16 | **EFF_PACK**       | Effective packing factor.                                     |
|    15 | **UNSUP_FMT**      | Unsupported ALT_FMT encountered; base mode used.              |
|    14 | **DOWNCAST_TAKEN** | Safe down-cast performed in hardware.                         |
|    13 | **SAT_HIT**        | Saturation event occurred under the current effective state.  |
|    12 | **FTZ_HIT**        | Subnormal flush-to-zero occurred under the current state.     |
|    11 | **EFF_ZMODE**      | Mirror effective masked-lane policy.                          |
|    10 | **EFF_SAE**        | Mirror effective SAE.                                         |
|   9:8 | **EFF_NAN_POL**    | Mirror effective NaN policy.                                  |
|   7:4 | **IE_MASK**        | Latched view of `EXC.EN` after SAE gating.                    |
|   3:0 | **RSV**            | Reserved (may include counters).                              |

`CAP.PREC.STAT` reflects the effective state after the most recent APPLY write. `UNSUP_FMT`, `SAT_HIT`, and `FTZ_HIT` are sticky until the next APPLY or reset. Reset defaults are `PET=FP16`, `EW=16`, `ACCW=FP32`, `SAT=0`, `ALT_EN=0`.

##### 4.4.2.5 FP Exception Control - trap/SAE precedence

| CSR               | Addr  | Access | Description                                                                  |
| ----------------- | ----- | ------ | ---------------------------------------------------------------------------- |
| `CAP.PREC.EXC.EN` | 0x7D3 | RW     | FP exception enables: `NV,DZ,OF,UF,NX` (bits 4:0). Trap only if `SAE_DEF=0`. |
| `CAP.PREC.EXC.ST` | 0x7D4 | RO/W1C | Sticky flags `NV,DZ,OF,UF,NX` + `QNAN_SEEN,SNAN_SEEN`. Write 1 clears bit.   |

* Per-instruction SAE overrides (e.g., RSV `svon.fpctl`) take precedence over `CAP.PREC.MODE.SAE_DEF`.
* Traps for NV,DZ,OF,UF,NX are delivered only if the relevant bit is 1 in `EXC.EN` and the effective SAE is 0; otherwise stickies in `EXC.ST` are set and execution continues.
* Stickies are persistent until explicitly cleared by a W1C write; they are not auto-cleared by `APPLY0/1`.
* Units MUST set sticky bits when exceptions or NaN observations occur. If `CAP.PREC.MODE.SAE_DEF=1` or a per-op SAE override is set, units suppress synchronous traps but still update stickies.

> **RSV note (informative):** A one-shot prefix (e.g., `svon.fpctl`) may override **RC/SAE/ZMODE** for the next instruction without touching CAP. If absent, software can adjust the defaults via `CAP.PREC.MODE` between operations.

### 4.5 Precision/Format Capability Descriptors (RO, non-binding)

These CSRs report supported data formats per domain and provide generic format capability summaries. They are advisory and do not modify numeric policy.

**Normative requirements:**
1. All CSRs in this subsection MUST be read-only.
2. Bits not defined in Tables 4.5.1 and 4.5.2 MUST read as zero and MUST be ignored by software.
3. These CSRs MUST NOT alter precision or numeric policy; they only advertise implementation support.

#### 4.5.1 Per-domain format sets

These CSRs are read-only capability bitmasks that enumerate formats and typed operations supported by each pipeline domain (ALU, texture, image/buffer, RT). They are descriptive and non-binding.

| CSR            | Addr  | Access | Description                                      |
| -------------- | ----- | ------ | ------------------------------------------------ |
| `CAP.PREC.ALU` | 0x7D8 | RO     | ALU data types present (FP32/FP16/BF16/INT* variants). |
| `CAP.PREC.TEX` | 0x7D9 | RO     | Texture formats supported.                       |
| `CAP.PREC.IMG` | 0x7DA | RO     | Image/buffer formats & typed atomic support.     |
| `CAP.PREC.RT`  | 0x7DB | RO     | RT payload/barycentric format support.           |

**Informative:** Per-domain capability bitmasks describe which formats and operations are available in each pipeline (ALU, texture, image/buffer, RT). They do not change behavior.

#### 4.5.2 Generic precision capability summary

These CSRs are read-only summary descriptors for aggregate precision capabilities across domains. They are descriptive and non-binding.

| CSR             | Addr  | Access | Description                                      |
| --------------- | ----- | ------ | ------------------------------------------------ |
| `CAP.PREC.CAP`  | 0x7DC | RO     | Summary of supported PET/EW/ACCW combinations.   |
| `CAP.PREC.FP8`  | 0x7DD | RO     | FP8 variants (E4M3/E5M2/BF16-lite compatibility).|
| `CAP.PREC.INTQ` | 0x7DE | RO     | Quantized integer formats (INT4/INT2, dotprod).  |
| `CAP.PREC.HYB`  | 0x7DF | RO     | Hybrid / dual-precision capabilities.            |

* `CAP.PREC.CAP` may encode, for example, which `(PET,EW,ACCW)` triplets are natively supported vs emulated.
* `CAP.PREC.FP8` focuses on FP8-like DL formats and their relationships to BF16/FP16.
* `CAP.PREC.INTQ` focuses on low-bit INT quantization (INT8/4/2, dot-product, per-channel scale support, etc.).
* `CAP.PREC.HYB` captures mixed configurations (e.g. FP8 input / FP16 acc / FP32 resolve; FP16+INT8 hybrid pipes).

**Informative:** These summaries indicate supported combinations and relationships (native vs emulated PET/EW/ACCW sets, FP8 variants vs BF16/FP16 proximity, low-bit INT quantization, hybrid pipelines). They are advisory; software may ignore them.

Read-only capability CSRs describe supported data formats; they do not alter numeric policy.

* `CAP.PREC.ALU`: bits for `HAS_FP32`, `HAS_FP16`, `HAS_BF16`, `HAS_INT32/16/8`, `HAS_DOTPROD8` (if applicable), etc.
* `CAP.PREC.TEX`: support for `UNORM8/16`, `SNORM8/16`, `SRGB8_A8`, `FP16`, `FP32`, and optionally block-compressed families (BC1-7) as feature clusters.
* `CAP.PREC.IMG`: classes `R8/R16/R32_{U|S|F}`, `RG*`, `RGBA*`, typed-atomic flag(s).
* `CAP.PREC.RT`: payload/barycentrics: `PAYLOAD_FP32`, `PAYLOAD_FP16`, `BARY_FP32`, `BARY_FP16`, etc.

Undefined bits read zero and MUST be ignored by software. These CSRs are advisory and do not change architectural behavior.

TODO-ELABORATE: Add bitfield definitions for CAP.PREC.{ALU,TEX,IMG,RT} and CAP.PREC.CAP/FP8/INTQ/HYB.

### 4.6 Memory & Gather/Scatter Policy (architectural state)

These CSRs centralize memory classes, coherence domains, streaming defaults, descriptor policy, and indexed gather/scatter so all XPHMG domains (RSV, MTX, GFX, RT, and others) observe a single architectural view.

**Normative requirements:**
1. CSRs in this subsection define architectural state consumed by every XPHMG domain; implementations MUST apply the effective CAP.XMEM and SVMEM state at each instruction boundary unless a per-job or per-instruction override explicitly supersedes it.
2. Fields marked RO MUST ignore writes; fields marked RO/W1C MUST clear only the bits written as 1 and ignore zeros; RW fields MUST accept writes. Writes to fields tied to an unsupported feature (per `CAP.XMEM.CAP`) MUST be ignored and read zero.
3. Bits not defined in Tables 4.6.1-4.6.3 MUST read as zero and MUST be ignored on write.
4. Changes to CAP.XMEM and SVMEM take effect no earlier than the next instruction boundary; in-flight instructions execute with the state latched at decode.

#### 4.6.1 Capability & static parameters (Read-Only)

These CSRs expose static capabilities and size/limit parameters for the memory subsystem.

| CSR               | Addr  | Access | Description                                                                                                        |
| ----------------- | ----- | ------ | ------------------------------------------------------------------------------------------------------------------ |
| `CAP.XMEM.CAP`    | 0x7E0 | RO     | Feature bits: `LDS,L1,L2,XDMA,WAY_PART,LINE_LOCK,NONTEMP,SECT,COMPRESS,CLASS_PART,STREAM_PROF,SEC_DOM,BIDIR_SWZ`.  |
| `CAP.XMEM.PARAMS` | 0x7E1 | RO     | Encodes sizes/limits: `{LDS_KiB[15:0], L1_LINE_LOG2[19:16], L1_WAYS[27:20], MAX_SPROF[31:28]}` (extend as needed). |

#### 4.6.2 Policy state (Read/Write) - defaults all engines must honor

These CSRs hold architectural defaults for memory classes, streaming, coherence, compression, and descriptors. All domains MUST honor them unless overridden by a domain-specific control or descriptor.

| CSR                   | Addr  | Access | Description                                                                                        |
| --------------------- | ----- | ------ | -------------------------------------------------------------------------------------------------- |
| `CAP.XMEM.L1CFG`      | 0x7E2 | RW     | `{WAYS_HINT[7:0], LINE_SZ[15:8], REPL[19:16], LOCK_EN[20], RSV[23:21], ACTIVE_CLASS[31:24]}`       |
| `CAP.XMEM.CLSMAP`     | 0x7E3 | RW     | Class mapping masks/weights for CL0..CL3 (way-mask + optional QoS weights).                        |
| `CAP.XMEM.LDS_BUDGET` | 0x7E4 | RW     | Per-class soft LDS budgets (KiB) for CL0..CL3.                                                     |
| `CAP.XMEM.L2CFG`      | 0x7E5 | RW     | `{VICT[3:0], QoS[7:4], SNC[8], COMP_THR[15:9]}`                                                    |
| `CAP.XMEM.DOM`        | 0x7E6 | RW     | Default coherence domain: `1=CPU, 2=RT, 3=MTX, 15=ALL`.                                            |
| `CAP.XMEM.STREAM_CTL` | 0x7E7 | RW     | Streaming detector defaults: `{stride,pf_depth,nontemp_en}`.                                       |
| `CAP.XMEM.SPROF0`     | 0x7E8 | RW     | Streaming profile #0 (stride/prefetch/nt policy).                                                  |
| `CAP.XMEM.SPROF1`     | 0x7E9 | RW     | Streaming profile #1.                                                                              |
| `CAP.XMEM.SPROF2`     | 0x7EA | RW     | Streaming profile #2.                                                                              |
| `CAP.XMEM.SPROF3`     | 0x7EB | RW     | Streaming profile #3.                                                                              |
| `CAP.XMEM.COMP_CTL`   | 0x7EC | RW     | Inline compression defaults (mode/level/dictionary id).                                            |
| `CAP.XMEM.DESCPOL`    | 0x7ED | RW     | Descriptor defaults: `{DEF_CLASS, HYB_ENABLE, HYB_BORROW_QOS, STRICT_DOM}`.                        |
| `CAP.XMEM.EVENTS`     | 0x7EE | RO/W1C | Sticky memory events: `INV_MISS,LOCK_OVF,LDS_OOM,DMA_ERR,COMP_ERR,BUDGET_OVF,SEC_VIOL,HYB_FALLBK`. |

> Informative: Address 0x7EF is intentionally left free for future growth in the XMEM group.

#### 4.6.3 Index-based gather/scatter (SVMEM) - architectural state (used by RSV, GFX, MTX, RT)

These CSRs provide default parameters for indexed gather/scatter (SVMEM). They are architectural and apply to all domains that issue indexed memory operations.

**Normative requirements:**
1. SVMEM operations MUST honor predication: masked-off lanes (per effective predication (PMASK, per selected `pbank`)) MUST NOT issue memory accesses.
2. Masked-lane writeback MUST follow `CAP.PREC.MODE.ZMODE` (merge vs zeroing) unless a per-instruction override is provided elsewhere.
3. Fault-Only-First (FOF): if `FOF=1`, the first faulting element MUST store its index in `SVFAULTI` (as defined in RSV) and the operation MUST abort; a retry may resume from that index.
4. RW fields in Table 4.6.3 MUST be applied at instruction boundary. Writes to fields tied to unsupported features MUST be ignored and read as zero.

| CSR                | Addr  | Access | Description                                                                                                            |
| ------------------ | ----- | ------ | ---------------------------------------------------------------------------------------------------------------------- |
| `CAP.XMEM.SVMBASE` | 0x7F0 | RW     | Base pointer for indexed addressing (`base`).                                                                          |
| `CAP.XMEM.SVMIDX`  | 0x7F1 | RW     | Register base (or memory pointer) holding indices/offsets for current op window.                                       |
| `CAP.XMEM.SVMSCL`  | 0x7F2 | RW     | Scale & mode: `{SCALE(1,2,4,8,16), SIGNED(1b), GATHER/SCATTER(1b), FOF(1b)}`.                                          |
| `CAP.XMEM.SVMLEN`  | 0x7F3 | RW     | Element byte length if different from `EFF_EW` (0 = derive from precision).                                            |
| `CAP.XMEM.SVMMODE` | 0x7F4 | RW     | Misc flags: `{BOUND_CHECK, ALLOW_OOB_ZERO, COMPRESS_EN, EXPAND_EN}` (unsupported bits are ignored; reads return zero). |

**Informative:**
- These CSRs establish architectural defaults; drivers or descriptors may override fields (e.g., per-job class/stream profile/domain). Absent overrides, engines use the CAP-provided defaults.
- If a feature bit in `CAP.XMEM.CAP` is 0, writes to its associated RW fields are ignored and reads return zero.

TODO-ELABORATE: Provide full bit/field layouts for CAP.XMEM.* and SVMEM CSRs, matching precision detail level.

---

## 5. Architectural Rules & Conformance

**Conceptual model (informative):**
* This section collects cross-domain architectural guarantees that bind all XPHMG engines to a single CAP-defined numeric and memory contract.
* Numeric policy changes are gated by APPLY (PREC) and become effective only after latching; memory/SVMEM state changes are visible at instruction boundaries without APPLY.
* Advisory CSRs remain descriptive; only architectural state (`CAP.PREC.*`, `CAP.XMEM.*`, SVMEM defaults) governs correctness.

### 5.1 Cross-Domain Interoperability & Implementation Requirements (Normative)

This section defines mandatory cross-domain guarantees for all XPHMG domains (RSV, MTX, RT, GFX, XMEM, NPU). These requirements are architectural and apply unless explicitly marked as Informative.

**Normative requirements:**
1. All domains MUST consume CAP-latched precision and memory state at the instruction boundary and MUST NOT observe mid-instruction CSR updates.
2. No domain MAY reinterpret CAP-defined numeric or memory semantics in a domain-specific way.
3. Debug/replay visibility of CAP state MUST reflect the effective state used for execution.

#### 5.1.1 Instruction-Boundary State Latching

Every XPHMG engine MUST latch the effective `CAP.PREC.*`, `CAP.XMEM.*`, and SVMEM state at the instruction boundary (decode/issue) and execute deterministically with that snapshot. Mid-instruction CSR writes MUST NOT affect in-flight work.

#### 5.1.2 Unified Numeric Semantics Across Domains

Numeric semantics defined by CAP (precision, rounding, saturation, NaN policy, quantization) MUST be applied identically across domains. Domains MUST NOT reinterpret CAP numeric fields differently.

#### 5.1.3 Uniform Mask & Predication Semantics

Masked operations in any domain that implements predication MUST derive the effective predicate from PMASK (per selected `pbank`) and MUST apply `CAP.PREC.MODE.ZMODE` to predicate-false *register-like* destinations (merge vs zero). Predicated memory operations (including indexed gather/scatter) MUST NOT issue memory accesses for predicate-false elements/lanes.

#### 5.1.4 FP Exception & Sticky Consistency

FP exception enables and stickies (`CAP.PREC.EXC.*`) MUST be honored uniformly. SAE defaults and per-instruction overrides apply as defined in the PREC section; stickies reflect all domains’ observations.

#### 5.1.5 Unified Memory Semantics (CAP.XMEM -> All Domains)

CAP-provided memory defaults (`CAP.XMEM.*`, SVMEM) MUST be consumed by RSV, GFX, MTX, RT, and other engines. Domain-local mirrors (if any) are micro-architectural only and MUST NOT alter semantics.

#### 5.1.6 Cross-Domain Dataflow Guarantees

Cross-domain dataflow MUST rely on architecturally visible tiers/LDS/class/coherence semantics. Private register bypass between domains is non-architectural.

#### 5.1.7 Apply Semantics & Forbidden Partial-State Windows

`APPLY0/1` (PREC) MUST be treated as write-one-to-apply; staged writes without APPLY are non-architectural. Memory/SVMEM state changes are visible only at instruction boundaries. Software MUST NOT assume intermediate/staged values are visible.

#### 5.1.8 Debug/Replay Determinism

Debug/replay MUST expose the effective CAP state used for execution, including the latched `IE_MASK` captured on APPLY writes. Replays MUST reapply the same state before executing the captured sequence.

**Notes for implementers (informative):**
* When mixing domains, program CAP once per phase and avoid per-engine divergence; per-instruction overrides (if any) are one-shot and should not be relied upon for long-lived policy.
* Capture `CAP.PREC.STAT` and relevant `CAP.XMEM.*`/SVMEM CSRs during debug traces to ensure deterministic replay.

### 5.2 Conformance & Reset Defaults

This section states mandatory execution, latching, and reset behavior for `XPHMG_CAP`. No new semantics are introduced; it clarifies how existing rules apply across domains.

#### 5.2.1 Architectural observability & determinism (Normative)

* All `XPHMG_*` domains (RSV/MTX/GFX/RT/XMEM/ACCEL) MUST observe `CAP.PREC.*` and `CAP.XMEM.*` at the instruction boundary; in-flight instructions execute with the state captured at decode.
* Debug/replay visibility of CAP state MUST reflect the effective state used for execution (including IE_MASK latched by APPLY writes).
* Implementations MUST NOT reinterpret CAP-defined numeric or memory semantics in a domain-specific way.

#### 5.2.2 Advisory vs. architectural fields (Normative)

* `CAP.PREC.*` and `CAP.XMEM.*` are architectural state; correctness depends on them.
* `CAP.FEAT*`, `CAP.PREC.{ALU,TEX,IMG,RT}`, `CAP.TIER.*`, `CAP.HINT.*` are advisory (non-binding). Software MUST treat them as hints; absence of a bit does not forbid an alternate mechanism unless another spec section explicitly states so.
* Advisory values are stable after reset unless `CAP.FLAGS.RUNTIME_MUTABLE=1` (see 5.2.6).

#### 5.2.3 Unsupported/ignored fields (Normative)

* Writes to unsupported precision fields MUST be ignored and read as zero; precision downgrades/omissions MUST set `CAP.PREC.STAT.UNSUP_FMT` when they affect numeric formats.
* Writes to unsupported XMEM/SVMEM features MUST be ignored and read as zero; `UNSUP_FMT` MUST NOT be used for memory feature absences.
* Bits not defined in a table MUST read zero and ignore writes (see section 4 general rules).

#### 5.2.4 Apply and programming sequences (Normative)

* `CAP.PREC.MODE/APPLY0` and `CAP.PREC.ALT/APPLY1` are write-one-to-apply gates for the PREC group only. Writes without APPLY MAY be internally staged but are architecturally invisible.
* XMEM and SVMEM CSRs take effect at the next instruction boundary with no APPLY bit; engines use the latest CAP state visible at decode.
* Multi-CSR programming is allowed. The architectural state is what is read after the final write in the sequence. Partial programming MUST NOT change visible state until the APPLY (PREC) or the boundary (XMEM/SVMEM) rule is satisfied.

#### 5.2.5 Forward progress & stalling (Normative)

* Precision or XMEM state changes MUST NOT induce deadlock. Engines MAY stall without side effects until the programmed state is coherent; forward progress MUST resume once the state is applied.

#### 5.2.6 Interaction with profiles and runtime mutability (Normative)

* If `CAP.FLAGS.HAS_PROFILES=1`, profiles are discoverable via `CAP.PROF.*` and are descriptive; profiles MUST NOT alter architectural numeric or memory semantics. Only advisory tiers/hints may vary by profile, and only if `RUNTIME_MUTABLE=1`.
* If `RUNTIME_MUTABLE=0`, advisory CSRs are stable after reset unless software writes architectural state.

#### 5.2.7 Reset defaults (Normative)

* `CAP.PREC.MODE` resets to `ZMODE=0` (merge), `SAE_DEF=0`, `NAN_POL=propagate`; other bits reset per implementation defaults consistent with section 4.4.2.1.
* `CAP.PREC.ALT` resets with `ALT_EN=0` and ALT fields zero.
* `CAP.PREC.EXC.{EN,ST}` reset to zero (all traps disabled; stickies clear).
* `CAP.XMEM.*` RW fields reset to 0 meaning "implementation defaults", except `ACTIVE_CLASS=0`.
 * Advisory CSRs (`CAP.FEAT*`, `CAP.PREC.{ALU,TEX,IMG,RT}`, `CAP.TIER.*`, `CAP.HINT.*`) reset to implementation-defined discovery values.

#### 5.2.8 Undefined / reserved behavior (Normative)

* Writes to reserved bits/fields MUST be ignored and MUST NOT cause traps. Reads of reserved bits MUST return zero unless a subsection states read-undef.
* Behavior is UNDEFINED if software depends on staged-but-not-applied PREC state or assumes advisory bits imply correctness guarantees.

**Informative:** These rules align CAP observability across domains and ensure deterministic replay: numeric policy uses APPLY gates; memory policy uses boundary latching; advisory fields guide tuning only. Reset defaults provide a known base state for toolchains and firmware.

**Notes for implementers (informative):**
* Do not rely on implementation-specific defaults for correctness; always program required CAP state explicitly.
* If `HAS_PROFILES=1`, profiles remain descriptive; they MUST NOT be used as a substitute for programming architectural state.
* If optional features are absent (capability bit = 0), writes to dependent fields are ignored; software should gate programming on capability discovery.

---

## 6. Programming Sequences

This section gives minimal, deterministic programming sequences for CAP numeric policy. It does not introduce new behavior; it restates how to program and observe state using the APPLY gates and sticky reporting.

**Conceptual model (informative):**
* PREC programming uses write-one-to-apply gates (`APPLY0/1`) to atomically latch new numeric policy; staged writes without APPLY are non-architectural.
* XMEM/SVMEM programming does not use APPLY; writes take effect at the next instruction boundary. Use reads to confirm effective state.
* Per-instruction overrides (when provided by a domain ISA) are one-shot; they do not modify CAP state and should be used sparingly to avoid divergence between domains.

**Notes for implementers (informative):**
* Stage multi-field PREC writes, then issue a single APPLY to avoid transient states; read back `CAP.PREC.STAT` to confirm `EFF_*` and `IE_MASK`.
* Probe capability CSRs before selecting formats or features; unsupported ALT formats or precision combos are reported via `CAP.PREC.STAT.UNSUP_FMT`.
* For memory policy changes, program `CAP.XMEM.*`/SVMEM and ensure no in-flight work depends on prior defaults; changes latch at instruction boundary.

### 6.1 Program base numeric mode (Normative sequence)

Example: FP16 compute with FP32 accumulation, default rounding, merge masking, traps suppressed by default.

```asm
# CAP.PREC.MODE
li  t0, (0<<30)              # SAT=0 (no saturation)
    | (0b000<<27)            # FP_RMODE=RNE
    | (0b10<<22)             # ACCW=FP32
    | (0b011<<19)            # PET=FP16
    | (0b01<<17)             # EW=16
    | (0<<11)               # ZMODE=merge
    | (0<<10)               # SAE_DEF=0 (traps allowed if enabled)
    | (1<<31)                # APPLY0
csrw 0x7D0, t0               # CAP.PREC.MODE
```

Rules:
* All fields become architecturally visible only when written with `APPLY0=1`.
* Per-instruction overrides (if defined by the domain ISA) take precedence for that instruction only.

### 6.2 Optional alternate/mixed mode (Normative sequence)

Example: enable FP8 E4M3 with FP16 accumulation, mixed mode, ALT-specific saturation.

```asm
# CAP.PREC.ALT
li  t1, (1<<30)              # ALT_EN=1
    | (0b0010<<26)           # ALT_FMT=FP8_E4M3
    | (0b01<<24)             # ALT_ACCW=FP16
    | (1<<22)                # MIXED=1 (e.g., FP8->FP16/FP32 acc)
    | (1<<8)                 # ALT_SAT=1
    | (1<<31)                # APPLY1
csrw 0x7D1, t1               # CAP.PREC.ALT
```

Rules:
* ALT takes effect only after `APPLY1=1`.
* If ALT is unsupported, hardware must fall back to base mode and set `CAP.PREC.STAT.UNSUP_FMT`.

### 6.3 Inspect effective state (Normative observation)

```asm
csrr t2, 0x7D2               # CAP.PREC.STAT
```

`CAP.PREC.STAT` mirrors the effective merged state (post-APPLY), including `EFF_*`, sticky observations (`SAT_HIT`, `FTZ_HIT`), and latched `IE_MASK`.

### 6.4 Configure FP exception enables (Normative sequence)

Example: enable traps for OF and NV, others sticky-only (subject to SAE gating):

```asm
li  t3, ((1<<2) /*OF*/ | (1<<4) /*NV*/)
csrw 0x7D3, t3               # CAP.PREC.EXC.EN
```

Rules:
* Traps occur only if the enable bit is 1 and effective SAE=0. Stickies always update.
* Clear stickies via W1C to `CAP.PREC.EXC.ST`.

### 6.5 Dispatch work

Jobs dispatched to XPHMG domains (RSV/MTX/GFX/RT/XMEM/ACCEL) execute using the effective CAP state latched at decode. To change policy between jobs, reprogram CAP and apply (`APPLY0/1`) before dispatch.

**Informative:** Toolchains should stage writes then use a single APPLY to avoid transient states. For memory defaults (`CAP.XMEM.*`), no APPLY bit exists; writes become visible at the next instruction boundary per section 5. All in-flight operations continue under the prior CAP snapshot; the updated CAP state affects only subsequently issued instructions.

---

## 7. Rationale

This section is informative. It explains the design intent behind centralizing capabilities and numeric/memory policy in `XPHMG_CAP`, without altering architectural semantics.

**Conceptual model (informative):**
* Central authority: CAP provides a single architectural anchor for numeric policy (`CAP.PREC.*`), memory defaults (`CAP.XMEM.*`/SVMEM), and capability discovery. This reduces divergence across domains and toolchains.
* Separation of concerns: Architectural policy is read/write and affects correctness; advisory descriptors are read-only and guide tuning only. Profiles are descriptive selectors gated by `HAS_PROFILES` and do not change architectural semantics.
* Determinism: APPLY gates (PREC) and instruction-boundary latching (XMEM/SVMEM) enforce deterministic execution and replay across domains.

**Architectural guarantees & invariants (informative):**
* Single source of truth for numeric and memory policy across RSV, MTX, GFX, RT, XMEM, and ACCEL domains.
* No domain-specific reinterpretation: downstream extensions consume CAP state; engine-local mirrors, if any, are micro-architectural.
* Unsupported/absent features read as zero and ignore writes; numeric format fallbacks are flagged via `CAP.PREC.STAT.UNSUP_FMT`.

**Interaction with other extensions (informative):**
* RSV/GFX/RT/MTX rely on CAP for default numeric/memory behavior; one-shot overrides (if provided by those ISAs) do not mutate CAP.
* XMEM uses `CAP.XMEM.*` as the architectural interface; engine-local controls mirror but do not replace CAP semantics.
* Profiles (if present) may co-vary advisory tiers/hints; they MUST NOT alter CAP-defined correctness semantics.

**Evolvability and tooling:**
* Versioning (`CAP.VERS`) and reserved fields allow evolution without breaking existing software; unimplemented bits read zero and ignore writes.
* A single CAP map supports deterministic capability probing, simplifying firmware bring-up, compiler selection of formats, and replay/debug capture.

NOTE: This rationale does not introduce new requirements; it summarizes intent and the expected behavior already defined in Sections 1–6.

---

## 8. Glossary of the new/changed bits

This section is informative. It restates key fields introduced or clarified by `XPHMG_CAP` and points to their authoritative definitions.

**Conceptual model (informative):** Each glossary item summarizes a field’s architectural role and where it is defined; see Section 4 for normative bit layouts.

* **`ZMODE`** (`CAP.PREC.MODE` bit 11): Default masked-lane writeback policy (merge vs zeroing) for domains that implement masking (RSV vectors, and any domain that adopts the same mask semantics). Per-instruction overrides, if defined by the domain ISA, take precedence for that instruction. Defined normatively in 4.4.2.1.
* **`SAE_DEF`** (`CAP.PREC.MODE` bit 10): Default suppress-all-exceptions for FP operations. Per-instruction SAE overrides, when present, supersede it. Traps occur only when the relevant enable bit is set *and* effective SAE is 0. Defined normatively in 4.4.2.1 and 4.4.2.5.
* **`NAN_POL`** (`CAP.PREC.MODE` bits 9:8): Global NaN propagation/squash policy (see 4.4.2.1). Applies uniformly across XPHMG domains using CAP numeric policy.
* **`CAP.PREC.EXC.{EN,ST}`**: FP exception enables and stickies (NV, DZ, OF, UF, NX plus NaN-seen flags). Stickies are W1C and persist across APPLY unless explicitly cleared. Defined normatively in 4.4.2.5.
* **`CAP.PREC.CAP*`** (`CAP.PREC.CAP`, `.FP8`, `.INTQ`, `.HYB`): Advisory capability summaries for supported precision/format combinations. Defined in 4.5.2.
* **`CAP.PREC.{ALU,TEX,IMG,RT}`**: Advisory per-domain format capability descriptors. Defined in 4.5.1.
* **`CAP.XMEM.*`**: Architectural memory defaults (classes, coherence, streaming, compression, descriptor policy) and SVMEM indexed gather/scatter defaults; consumed by all domains at instruction boundary. Defined normatively in 4.6.
* **`IE_MASK`** (`CAP.PREC.STAT` bits 7:4): Latched view of effective FP exception enables after SAE gating, captured on APPLY0/1 for debug/replay determinism. Defined normatively in 4.4.2.4.
* **`CAP.TIER.*` / `CAP.HINT.*`**: Advisory tier capacity and implementation hints. Defined in 4.2.2; conceptual model in 2.2.
* **`CAP.FEAT*`**: Advisory feature discovery bits. Defined in 4.2.1.
* **`CAP.PROF.*`**: Profile discovery CSRs (descriptive only, gated by `HAS_PROFILES`). Defined in 4.3.

### 8.1 Cross-references to other specs (editorial)

* Update **RSV** doc to state: predication zero/merge behavior is governed by `CAP.PREC.MODE.ZMODE`; per-op overrides are allowed if the optional `svon.fpctl` prefix is implemented.
* Update **XMEM** doc: its per-engine "programming" section should reference these CAP CSRs as the architectural sources of truth; engine-local mirrors (if any) are micro-architectural.
* Update **GFX/MTX/RT** docs: when describing memory behavior, point to `CAP.XMEM.DOM`, `.L1CFG`, `.CLS*`, and SVMEM controls for gather/scatter phases.

---
**License:** CC-BY-SA 4.0 - Open Specification maintained by Pedro H. M. Garcia.
