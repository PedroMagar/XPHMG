# RISC-V Vendor Extension **XPHMG_CAP**

**Capabilities, Numeric Policy & Precision**

**Category:** Vendor Extension (`XPHMG_CAP`)
**Version:** 0.1.0
**Author:** Pedro H. M. Garcia
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview

`XPHMG_CAP` defines the **authoritative architectural state** for the entire XPHMG family.

It unifies two distinct roles:

1. **Capability discovery** — read-only, non-binding descriptors that expose what the implementation can support; and
2. **Architectural numeric and memory policy** — mandatory state that governs how all XPHMG domains execute.

All numeric behavior (precision, element width, accumulator width, rounding, saturation, quantization, NaN policy, exceptions) and all memory policy (classes, coherence domains, streaming, gather/scatter) are owned exclusively by `XPHMG_CAP`.

No other XPHMG extension may redefine, override, or bypass CAP-defined semantics.

At each instruction boundary, every XPHMG engine must snapshot the effective CAP state and execute deterministically under that configuration until completion.

---

## 2. CSR Map (suggested addresses)

### 2.0 High-level XPHMG CSR layout

To avoid collisions between XPHMG sub-extensions, the vendor CSR window `0x7C0–0x7FF`
is partitioned as follows (all CSRs are XLEN-wide):

| Range          | Owner / Block          | Contents                                                    |
| -------------- | ---------------------- | ----------------------------------------------------------- |
| `0x7C0–0x7CF`  | `XPHMG_CAP` (core)     | ID, version, global flags, profiles, features, tiers, hints |
| `0x7D0–0x7D7`  | `XPHMG_CAP.PREC`       | Numeric/precision policy & FP exception CSRs                |
| `0x7D8–0x7DF`  | `XPHMG_CAP.PREC.CAP`   | Precision/format capability descriptors (see §2.5)          |
| `0x7E0–0x7EF`  | `XPHMG_CAP.XMEM`       | Memory / LDS / streaming / descriptor policy                |
| `0x7F0–0x7F7`  | `XPHMG_CAP.XMEM.SVM`   | Indexed gather/scatter (SVMEM) configuration                |
| `0x7F8–0x7FF`  | `XPHMG_RSV`            | Simple-Vectors control CSRs (see XPHMG_RSV)                 |

This document normatively defines the `XPHMG_CAP`-owned ranges; `XPHMG_RSV` defines
the semantics of its own CSRs within the reserved `0x7F8–0x7FF` sub-window.

### 2.1 Core Identification & Versioning (RO)

| CSR            | Addr  | Access | Description                                                  |
| -------------- | ----- | ------ | ------------------------------------------------------------ |
| `CAP.ID`       | 0x7C0 | RO     | Vendor/arch ID (e.g., 0x50484D47 “PHMG”).                    |
| `CAP.VERS`     | 0x7C1 | RO     | Layout version (Major:8, Minor:8, Patch:8, Rsvd:8).          |
| `CAP.FLAGS`    | 0x7C2 | RO     | Global flags (see §3.1).                                     |
| `CAP.PROF.CUR` | 0x7C3 | RO     | Current runtime profile (optional; valid if HAS_PROFILES=1). |
| `CAP.PROF.SUP` | 0x7C4 | RO     | Supported profiles bitmask (optional).                       |

> If you prefer, you may keep `CAP.PROF.*` in §5 and let 0x7C3/0x7C4 read zero when profiles are not implemented.

### 2.2 Feature Bits (RO, non-binding)

| CSR         | Addr  | Access | Description                                     |
| ----------- | ----- | ------ | ----------------------------------------------- |
| `CAP.FEAT0` | 0x7C5 | RO     | General features (bindless, OOO mem, RT, etc.). |
| `CAP.FEAT1` | 0x7C6 | RO     | Sync/barrier features, PMU presence.            |

### 2.3 Tiers & Hints (RO, non-binding)

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

### 2.4 Precision & Numeric Policy (architectural state)

| CSR               | Addr  | Access | Description                                                           |
| ----------------- | ----- | ------ | --------------------------------------------------------------------- |
| `CAP.PREC.MODE`   | 0x7D0 | RW     | Base precision & numeric policy.                                      |
| `CAP.PREC.ALT`    | 0x7D1 | RW     | Alternate/extended format control.                                    |
| `CAP.PREC.STAT`   | 0x7D2 | RO     | Effective state + stickies.                                           |
| `CAP.PREC.EXC.EN` | 0x7D3 | RW     | FP exception enables: NV,DZ,OF,UF,NX (bits 4:0).                      |
| `CAP.PREC.EXC.ST` | 0x7D4 | RO/W1C | Sticky flags NV,DZ,OF,UF,NX + QNAN_SEEN,SNAN_SEEN (write 1 to clear). |
| `CAP.PREC.RSV0`   | 0x7D5 | RO     | Reserved for future precision controls (reads zero).                  |

### 2.5 Precision/Format Capabilities (RO, non-binding)

This block describes which **data formats** are supported by the implementation,
both per-domain (ALU/TEX/IMG/RT) and in a generic, more fine-grained way
(FP8/INTQ/hybrid).

#### 2.5.1 Per-domain format sets

| CSR            | Addr  | Access | Description                                     |
| -------------- | ----- | ------ | ----------------------------------------------- |
| `CAP.PREC.ALU` | 0x7D8 | RO     | ALU data types present (FP32/FP16/BF16/INT* …). |
| `CAP.PREC.TEX` | 0x7D9 | RO     | Texture formats supported.                      |
| `CAP.PREC.IMG` | 0x7DA | RO     | Image/buffer formats & typed atomic support.    |
| `CAP.PREC.RT`  | 0x7DB | RO     | RT payload/barycentric format support.          |

These are **per-domain** capability bitmasks. They do not change behavior,
only describe which formats and operations are available in each pipeline.

#### 2.5.2 Generic precision capability summary

| CSR             | Addr  | Access | Description                                      |
| --------------- | ----- | ------ | ------------------------------------------------ |
| `CAP.PREC.CAP`  | 0x7DC | RO     | Summary of supported PET/EW/ACCW combinations.   |
| `CAP.PREC.FP8`  | 0x7DD | RO     | FP8 variants (E4M3/E5M2/BF16-lite compatibility).|
| `CAP.PREC.INTQ` | 0x7DE | RO     | Quantized integer formats (INT4/INT2, dotprod).  |
| `CAP.PREC.HYB`  | 0x7DF | RO     | Hybrid / dual-precision capabilities.            |

* `CAP.PREC.CAP` may encode, for example, which `(PET,EW,ACCW)` triplets are
  natively supported vs emulated.
* `CAP.PREC.FP8` focuses on FP8-like DL formats and their relationships to BF16/FP16.
* `CAP.PREC.INTQ` focuses on low-bit INT quantization (INT8/4/2, dot-product,
  per-channel scale support, etc.).
* `CAP.PREC.HYB` captures “mixed” configurations (e.g. FP8 input / FP16 acc /
  FP32 resolve; FP16+INT8 hybrid pipes).

All these CSRs are **advisory**: they do not change behavior, they only expose
what the hardware can actually do. Software is free to ignore them.

### 2.6 Memory & Gather/Scatter Control (architectural state)

These CSRs live in CAP so that **all domains** (RSV, MTX, GFX, RT) honor the same defaults and discovery.

#### 2.6.1 Capability & static parameters (Read-Only)

| CSR               | Addr  | Access | Description                                                                                                        |
| ----------------- | ----- | ------ | ------------------------------------------------------------------------------------------------------------------ |
| `CAP.XMEM.CAP`    | 0x7E0 | RO     | Feature bits: `LDS,L1,L2,XDMA,WAY_PART,LINE_LOCK,NONTEMP,SECT,COMPRESS,CLASS_PART,STREAM_PROF,SEC_DOM,BIDIR_SWZ`.  |
| `CAP.XMEM.PARAMS` | 0x7E1 | RO     | Encodes sizes/limits: `{LDS_KiB[15:0], L1_LINE_LOG2[19:16], L1_WAYS[27:20], MAX_SPROF[31:28]}` (extend as needed). |

#### 2.6.2 Policy state (Read/Write) — defaults all engines must honor

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
| `CAP.XMEM.EVENTS`     | 0x7EE | RW/W1C | Sticky memory events: `INV_MISS,LOCK_OVF,LDS_OOM,DMA_ERR,COMP_ERR,BUDGET_OVF,SEC_VIOL,HYB_FALLBK`. |

> 0x7EF intentionally left free for future growth in the XMEM group.

#### 2.6.3 Index-based gather/scatter (SVMEM) — architectural state (used by RSV, GFX, MTX, RT)

Add a compact set that **RSV gather/scatter** and any domain can consume:

| CSR                | Addr  | Access | Description                                                                                                            |
| ------------------ | ----- | ------ | ---------------------------------------------------------------------------------------------------------------------- |
| `CAP.XMEM.SVMBASE` | 0x7F0 | RW     | Base pointer for indexed addressing (`base`).                                                                          |
| `CAP.XMEM.SVMIDX`  | 0x7F1 | RW     | Register base (or memory pointer) holding indices/offsets for current op window.                                       |
| `CAP.XMEM.SVMSCL`  | 0x7F2 | RW     | Scale & mode: `{SCALE(1,2,4,8,16), SIGNED(1b), GATHER/SCATTER(1b), FOF(1b)}`.                                          |
| `CAP.XMEM.SVMLEN`  | 0x7F3 | RW     | Element byte length if different from `EFF_EW` (0 = derive from precision).                                            |
| `CAP.XMEM.SVMMODE` | 0x7F4 | RW     | Misc flags: `{BOUND_CHECK, ALLOW_OOB_ZERO, COMPRESS_EN, EXPAND_EN}` (unsupported bits are ignored; reads return zero). |

**Fault-Only-First (FOF):**

* If `FOF=1`, on the first faulting element the unit writes that element index into `SVFAULTI` (your existing `RSV` CSR) and aborts the op; a retry may resume from that index.

**Predication:**

* Gather/scatter MUST honor `SVPMASK*`. Masked-off lanes do not issue memory accesses.

**Zeroing vs merge (masked lanes):**

* Respect `CAP.PREC.MODE.ZMODE`: if zeroing, masked destinations receive zero; if merge, destinations are preserved.

**Semantics:**

* These CSRs establish **architectural defaults**. Drivers/descriptors may override (e.g., per-job `mclass/sprof_id/dom`), but absent overrides the engines MUST use these CAP values.
* If a hardware feature in `CAP.XMEM.CAP` is 0, writes to its associated RW fields are **ignored**, and reads return zero.

### **2.7 Cross-Domain Interoperability & Implementation Requirements (Normative)**

*(New section added to XPHMG_CAP — applies to **all** XPHMG domains: RSV, MTX, RT, GFX, XMEM, NPU)*


This section formalizes the architectural guarantees required for **correct and high-performance interaction** between all XPHMG domains.
All rules here are **mandatory**, unless explicitly marked as advisory.

#### **2.7.1 Instruction-Boundary State Latching**

Every XPHMG engine **must snapshot** the following CAP-controlled architectural state **at the instruction boundary**:

* `CAP.PREC.MODE`
* `CAP.PREC.ALT`
* `CAP.PREC.EXC.EN`
* All relevant fields of `CAP.XMEM.*` (memory classes, streaming, coherence domain, SVMEM parameters)

Once a domain has latched these values, **no mid-instruction modification** of any CAP CSR may affect the in-flight instruction.

**Rationale:** Guarantees determinism and prevents partial adoption of numeric or memory modes.

#### **2.7.2 Unified Numeric Semantics Across Domains**

All domains — including RSV, MTX, RT, GFX, scalar ALU, and any future XPHMG accelerators — must interpret:

* **PET, EW, ACCW**
* **rounding modes (`FP_RMODE`)**
* **saturation (`SAT` / `ALT_SAT`)**
* **quantization (`Q`, zero-point, scale-shift)**
* **NaN policies (`NAN_POL`)**
* **ZMODE (zero-or-merge masked lanes)**
* **SAE and exception mask (`IE_MASK`)**

in exactly the same architectural manner as defined in **CAP effective state (`CAP.PREC.STAT`)**.

**Consequence:**
A value produced by MTX must be consumed identically by RSV, GFX, RT, or XMEM gather/scatter without reinterpretation, reconfiguration, or domain-local variant rules.

#### **2.7.3 Uniform Mask & Predication Semantics**

All masked operations — including:

* RSV predicated ops,
* RSV gather/scatter,
* MTX vector/tile loads when aliasing RSV,
* RT/GFX vectorized lanes (optional),
* SVMEM indexed operations —

must apply **the same ZMODE behavior**:

* `ZMODE = 0` → masked-off elements **preserve the old value** (merge)
* `ZMODE = 1` → masked-off elements **write zero**

Per-instruction overrides (e.g., RSV `svon.fpctl`) **modify semantics for that instruction only**, without altering CAP defaults.

#### **2.7.4 FP Exception & Sticky Consistency**

All domains must update sticky bits in:

* `CAP.PREC.STAT`
* `CAP.PREC.EXC.ST`

using precisely the same rules for:

* `NV` (invalid)
* `DZ` (divide-by-zero)
* `OF` (overflow)
* `UF` (underflow)
* `NX` (inexact)
* `QNAN_SEEN`
* `SNAN_SEEN`
* `SAT_HIT`, `FTZ_HIT`, `DOWNCAST_TAKEN`, `UNSUP_FMT`

No domain may introduce domain-specific FP exception behavior.

Traps occur only if:

1. The corresponding bit in `CAP.PREC.EXC.EN` is **1**, and
2. Effective SAE (`EFF_SAE`) is **0**.

Otherwise the event is recorded in sticky flags.


#### **2.7.5 Unified Memory Semantics (CAP.XMEM → All Domains)**

All memory-touching instructions — including:

* scalar loads/stores,
* RSV loads/stores,
* RSV gather/scatter,
* MTX tile loads/stores,
* RT BVH & triangle loads,
* GFX texture/pixel loads,
* XMEM descriptor jobs —

must:

1. **Obey the same coherence domain** (`CAP.XMEM.DOM`).
2. **Respect the same memory class mapping** (`CAP.XMEM.CLSMAP`).
3. **Adopt the same streaming & prefetch parameters** (`STREAM_CTL`, `SPROF*`).
4. **Use identical SVMEM rules** (index scaling, predication, FOF, masked-lane ZMODE).
5. Follow `CAP.XMEM.COMP_CTL` for inline compression defaults.
6. Apply descriptor defaults (`DESCPOL`) unless overridden per-job.

No domain may implement memory semantics that diverge from CAP-defined XMEM state.


#### **2.7.6 Cross-Domain Dataflow Guarantees**

Because domains share CAP numeric and XMEM memory state:

* **RSV → MTX:** RSV permutes/vec ops may prepare data directly for MTX without reformatting.
* **MTX → RSV:** MTX results (tiles/vectors) must be readable by RSV under the same precision/mask rules.
* **RSV → RT/GFX:** Shading, barycentric ops, or SoA/AoS transforms behave identically under CAP precision.
* **MTX → RT:** MTX transforms for instancing (matrix/vector) require no precision-mapping differences.
* **RT/GFX → RSV/MTX:** Visibility results and shading data must obey the same SVMEM and ZMODE semantics.
* **All → XMEM:** All memory access—including LDS-class borrowing—must respect `CAP.XMEM.*`.

This ensures no ISA feature becomes redundant: each domain addresses distinct execution granularities (vector, tile, ray, shading) while remaining interoperable.


#### **2.7.7 Apply Semantics & Forbidden Partial-State Windows**

* Writes to `CAP.PREC.MODE` and `CAP.PREC.ALT` only take effect when `APPLY0=1` or `APPLY1=1`.
* Partial writes without APPLY **must not** affect execution.
* While APPLY is pending, **other domains must not observe intermediate state**.
* XMEM/SVMEM CSRs take effect on the **next instruction boundary** (no APPLY bits).

This prevents timing-dependent behavior and implementation divergence.

#### **2.7.8 Debug/Replay Determinism**

Implementations must ensure:

* `CAP.PREC.STAT` always exposes the **effective** state used by the decoded instruction stream.
* Debuggers see a **stable, latched** view of precision/memory settings.
* Replay engines produce identical results when executing the same CSR sequence.


### 2.8 Tiered Execution and Shared Hot Data Model (Global Policy)

`XPHMG` defines a unified memory and dataflow model for all accelerators (RSV, MTX, GFX, RT, ACCEL).  
Communication between engines MUST NOT rely on direct cross-pipeline register forwarding.  
Instead, all inter-engine dataflow occurs through a tiered memory hierarchy managed by `CAP.XMEM.*` and `XPHMG_MEMCTL`.

#### 2.8.1 Tier Model (Architectural)

The implementation MUST expose at least the following conceptual tiers:

| Tier | Description |
|------|-------------|
| **Tier-0 (T0)** | A small, low-latency region of LDS/XMEM treated as *quasi-register* storage. Used for hot data passed between engines. |
| **Tier-1 (T1)** | Normal LDS/XMEM scratchpad. Larger, medium-latency, used for per-stage working sets. |
| **Tier-2 (T2)** | Coherent caches / shared memory hierarchy. |
| **Tier-3 (T3)** | DRAM or external memory. |

Tier-0 is the **unique** architecturally visible level where cross-engine handoff occurs **without data movement**.  
Private engine register files are not visible architecturally and are outside the scope of this specification.

#### 2.8.2 Global Tier Capabilities (`CAP.TIER`)

A `CAP.TIER` CSR SHALL describe:

- Supported tier count (min 2: T0 and T1).  
- Maximum Tier-0 capacity (in KB or in implementation quanta).  
- Whether Tier-0 is physically mapped inside LDS.  
- Whether Tier-0 content is visible to scalar/RSV execution.  
- Whether Tier-0 supports cross-engine shared access (`T0_SHARED=1`).

#### 2.8.3 Global Dataflow Policy

Whenever multiple engines participate in a pipeline (e.g., MTX→RT→GFX), all data exchanges MUST occur through:

- **Tier-0** when low-latency, hot-region communication is required.  
- **Tier-1** as fallback when Tier-0 capacity is insufficient.

Engine-to-engine transfers MUST NOT imply physical data copies.  
Changing logical ownership (`dom`) of a Tier-0 region constitutes the architectural definition of a handoff.

#### 2.8.4 Tier-1 Working Set Policy

Tier-1 (T1) represents the primary *working set* level of the XPHMG memory hierarchy.  
It is implemented using LDS/XMEM scratchpad and/or low-latency local memory, and is visible to all engines (scalar/RSV, MTX, GFX, RT, ACCEL).

Tier-1 is **not** a replacement for Tier-0:

- Tier-0 (T0) is reserved for *hot, cross-engine handoffs* of quasi-register data (`CLS_SHARED / T0`).
- Tier-1 (T1) is used to build, transform and stage data structures that may later be consumed by Tier-0, by other engines, or by DRAM.

Architecturally:

- Engines MUST treat Tier-1 as the **default target** for local working sets that:
  - Do not require lowest possible latency (Tier-0),
  - Do not require global coherence (Tier-2+),
  - Are intended to be reused across multiple instructions or micro-passes.

- Cross-engine communication that does **not** require hot, low-latency handoff MAY occur via Tier-1 regions, using normal XMEM ordering and synchronization rules.

- The CAP tier model does **not** define data *format*; it defines **where** data resides and what latency/visibility guarantees apply. Data layout and tiling are handled by the XDL model and `XPHMG_MEMCTL` layout metadata (see XMEM spec).

##### 2.8.4.1 Tier-1 Capability Hints

`CAP.TIER` SHOULD expose implementation hints for Tier-1:

- Approximate aggregate Tier-1 capacity per hart/cluster.
- Whether Tier-1 is purely LDS, or may spill into higher levels transparently.
- Whether Tier-1 is physically partitioned per engine or shared.
- Recommended maximum working-set size per engine for optimal performance.

These hints are non-binding but allow runtimes and compilers to select appropriate staging strategies (e.g., BVH construction in RT, tile staging in MTX, G-buffer staging in GFX, tensor blocking for ACCEL).

#### 2.8.5 Tier-2 and Tier-3 Policy (Caches and External Memory)

Tier-2 (T2) represents the *coherent cache / shared-memory hierarchy* visible to XPHMG engines:

- T2 encompasses on-chip caches (e.g., L1/L2) and any fabric-level shared memory that participates in hardware coherence.
- T2 is not directly addressable as a separate address space; it is an attribute of how addresses are serviced.
- Correctness of XPHMG programs must not depend on whether data resides in T1, T2 or T3 at any given moment. Tiers are a performance and latency model.

Tier-3 (T3) represents *external, high-latency memory* (DRAM or equivalent):

- T3 is the ultimate backing store for all cacheable regions.
- Streaming engines (XDMA) and non-temporal XMEM operations typically target T3 as their primary sink/source.
- Implementations MAY expose memory-class or streaming-profile hints that bias allocation and replacement toward T2 or T3 residency; these hints are non-binding.

Architecturally:

- `tier = T2` in `XPHMG_MEMCTL` indicates that the region is intended to be serviced primarily through the coherent cache subsystem.
- `tier = T3` indicates that the region is expected to be accessed with long-latency, streaming-friendly paths (e.g., XDMA, non-temporal loads/stores).

Engine-visible behavior:

- Loads/stores to T2/T3 regions must obey the same coherence and ordering rules as regular CPU-visible memory.
- Fences and scopes (`THREAD < WARP < WG < CLUSTER < SYSTEM`) apply equally to T2/T3 accesses, as defined by XMEM.
- XPHMG does not standardize specific cache sizes, associativity or replacement policies; these are implementation-defined.

#### 2.8.6 Tile Capability Enumeration (`CAP.TILE`)

`CAP.TILE` is a read-only CSR that describes MTX tile support and preferred composite sizes.

At minimum, any implementation that claims `XPHMG_MTX` MUST set `BASE4x4 = 1`, indicating support for the canonical `XDL_TILE4x4_ROWMAJOR` layout.

A suggested encoding is:

| Bits   | Name        | Description |
|--------|-------------|-------------|
| 0      | BASE4x4     | MUST be 1 if `XPHMG_MTX` is implemented. Indicates support for the canonical 4×4 tile (`XDL_TILE4x4_ROWMAJOR`). |
| 1      | COMPOSITE   | If 1, the implementation supports composite tiles formed by linking multiple 4×4 tiles as defined by the XDL model. Software may request M×N tiles where M and N are multiples of 4, up to the limits encoded by `MAX_M_LOG2` and `MAX_N_LOG2`. |
| 2      | WIDE_NATIVE | If 1, the implementation has native hardware support for at least one tile shape larger than 4×4 (e.g., 4×8, 8×8, 16×16), instead of always decomposing them into multiple 4×4 sub-operations. |
| 4:3    | MAX_M_LOG2  | Encodes the maximum supported logical tile row count **M** as `M = 4 << MAX_M_LOG2`. Value 0 ⇒ M=4, 1 ⇒ M=8, 2 ⇒ M=16. Value 3 is reserved for future extensions. |
| 6:5    | MAX_N_LOG2  | Encodes the maximum supported logical tile column count **N** as `N = 4 << MAX_N_LOG2`. Same encoding as `MAX_M_LOG2`. |
| 31:7   | Reserved    | Must be zero. |

Semantics:

- `BASE4x4` MUST be 1 for any conformant `XPHMG_MTX` implementation.

- If `COMPOSITE = 0`, software MUST assume that only 4×4 tiles are supported as single-tile MTX operands; larger shapes MUST be implemented in software using multiple MTX operations over 4×4 tiles.

- If `COMPOSITE = 1`, software MAY treat any M×N shape where:
  - M and N are multiples of 4, and
  - `M ≤ 4 << MAX_M_LOG2`, `N ≤ 4 << MAX_N_LOG2`

  as a valid composite tile shape for MTX operations and helpers (MAC, transpose, epilogue, reductions), with the understanding that the implementation may internally decompose such tiles into multiple 4×4 sub-operations.

- `WIDE_NATIVE` is a performance hint:
  - If `WIDE_NATIVE = 1`, the implementation is expected to have at least one native tile shape larger than 4×4 (e.g., 4×8, 8×8, 16×16) for which it can execute composite operations with higher throughput than a naive sequence of 4×4 operations.
  - Software SHOULD use `MAX_M_LOG2`, `MAX_N_LOG2` and `WIDE_NATIVE` to pick preferred blocking (e.g., 8×8 or 16×16 micro-kernels) for a given target.

Notes:

- This encoding limits the architectural maximum to 16×16 tiles (M,N ≤ 16). Larger tiles (e.g., 32×32) may still be implemented as software-level compositions of smaller tiles, but are out of scope for this version of `XPHMG_MTX`.

---

## 3. Field Definitions

### 3.1 `CAP.FLAGS` (RO)

* `RUNTIME_MUTABLE` (bit 0): Some tiers/hints may vary by power/thermal profile.
* `HAS_PROFILES` (bit 1): Profile control/status is exposed (see §5).
* Other bits reserved.

### 3.2 `CAP.FEAT0` (RO)

Suggested bits (example mapping):

* `BINDLESS`, `OOO_MEM`, `DYN_REG_ALLOC`, `RT_PRESENT`, `LDS_ATOMICS`, `ANISO`, `IMG_ATOMICS`, `SLC_PRESENT`, `EXT_BARRIERS`, `PMU_PRESENT`.

### 3.3 Precision & Numeric Policy (architectural state)

#### 3.3.1 `CAP.PREC.MODE` — Base Precision Control (RW)

**Purpose:** default numeric type/behavior for GFX/ALU, MTX, RT, XMEM.

| Bits   | Name           | Description                                                                  |
| ------ | -------------- | ---------------------------------------------------------------------------- |
| XLEN-1 | **APPLY0**     | *Write-one-to-apply*. Latches fields atomically; auto-clears.                |
| 30     | **SAT**        | Saturate on conversion overflow (0 trap/default, 1 saturate).                |
| 29:27  | **FP_RMODE**   | FP rounding: 000 RNE, 001 RTZ, 010 RUP, 011 RDN, 100 RMM; else reserved.     |
| 26     | **DENORM_FTZ** | Flush subnormals to zero.                                                    |
| 25     | **Q**          | Enable quantized integer path (INT8/INT4 policies).                          |
| 24     | **UNS**        | Integers treated as unsigned in integer paths.                               |
| 23:22  | **ACCW**       | Accumulator width: 00=equal EW, 01=FP16, 10=FP32, 11=impl-def.               |
| 21:19  | **PET**        | Primary element type: 000 INT8, 001 INT16, 010 BF16, 011 FP16, 100 FP32.     |
| 18:17  | **EW**         | Element width: 00=8, 01=16, 10=32, 11=64.                                    |
| 16:12  | **PACK**       | Vector packing: 0=off, 1=INT4 nibble, 2=INT2; others reserved.               |
| 11     | **ZMODE**      | 0 = merge (masked lanes preserve rd), 1 = zeroing (masked lanes write zero). |
| 10     | **SAE_DEF**    | 1 = suppress synchronous FP exceptions by default.                           |
| 9:8    | **NAN_POL**    | 00 propagate, 01 quiet-NaN→0, 10 quiet-NaN→+INF (impl-def), 11 reserved.     |
| 7:0    | **RSV0**       | Reserved (future: scale presets/mixed-precision hints).                      |

> These are architectural defaults consumed by **RSV/MTX/GFX/XMEM**. Per-instruction overrides are allowed by the ISA (see RSV note below) but not required.

**Semantics**

* Writing with `APPLY0=1` commits all fields atomically; reads return the active state.
* If no alternate mode is active, this CSR fully defines execution.
* **SAT (bit 30):** When 1, conversions/clamps **saturate on writeback** to the representable range of `PET/EW`; when 0, conversions follow normal FP/INT rules and any overflows/underflows are reported only via `EXC.ST` (and may trap if enabled in `EXC.EN` and `SAE_DEF=0`).

#### 3.3.2 `CAP.PREC.ALT` — Alternate/Extended Mode (RW)

**Purpose:** selects compact/experimental numeric formats (FP8, INT4, INT2, BF16-Lite), mixed modes, and quantization.

| Bits   | Name            | Description                                                                                     |
| ------ | --------------- | ----------------------------------------------------------------------------------------------- |
| XLEN-1 | **APPLY1**      | *Write-one-to-apply*. Applies and auto-clears.                                                  |
| 30     | **ALT_EN**      | Enable alternate mode.                                                                          |
| 29:26  | **ALT_FMT**     | 0000 NONE, 0001 BF16_LITE, 0010 FP8_E4M3, 0011 FP8_E5M2, 0100 INT4, 0101 INT2; others reserved. |
| 25:24  | **ALT_ACCW**    | Accumulator override: 00=inherit MODE.ACCW, 01=FP16, 10=FP32, 11=impl-def.                      |
| 23     | **DOWNCAST_OK** | Allow safe down-cast (e.g., FP16→FP8).                                                          |
| 22     | **MIXED**       | Mixed precision (e.g., mul FP8 → acc FP16/32).                                                  |
| 21     | **ZP_EN**       | Enable zero-point quantization.                                                                 |
| 20:13  | **ZP**          | Zero-point value (8 effective bits recommended).                                                |
| 12:9   | **SCALE_SH**    | Log₂ scale shift (signed −8…+7).                                                                |
| 8      | **ALT_SAT**     | Alternate-specific saturation (priority over `SAT`).                                            |
| 7:3    | **POLICY1**     | Additional policy bits (impl-def).                                                              |
| 2:0    | **VER**         | Version/catalog index for the ALT format table.                                                 |

**ALT Mapping (normative)**

| ALT_FMT   | Effective PET | EW | Notes                                |
| --------- | ------------- | -- | ------------------------------------ |
| BF16_LITE | BF16          | 16 | Reduced exponent/mantissa precision. |
| FP8_E4M3  | FP8           | 8  | DL float.                            |
| FP8_E5M2  | FP8           | 8  | Transformer float.                   |
| INT4      | INT4          | 8  | 2× INT4 per byte; forces `PACK ≥ 1`. |
| INT2      | INT2          | 8  | 4× INT2 per byte; forces `PACK ≥ 2`. |

#### 3.3.3 Composition Rules (Effective State `EFF`)

* If `ALT_EN=0`: `EFF = CAP.PREC.MODE`.

* If `ALT_EN=1`: merge field-wise:

  * `PET, EW` ← ALT mapping table.
  * `ACCW` ← `ALT_ACCW!=00 ? ALT_ACCW : MODE.ACCW`.
  * `SAT`  ← `ALT_SAT ? 1 : MODE.SAT`.
  * `Q, UNS, FP_RMODE, DENORM_FTZ` ← inherit from `MODE`.
  * `PACK` ← inherit from `MODE`, except when ALT requires a minimum.
  * `ZP_EN/ZP/SCALE_SH` apply **only** if `MODE.Q=1`.
  * `EFF_ZMODE`, `EFF_SAE`, `EFF_NAN_POL` (mirror effective defaults in use).
  * `IE_MASK` (latched view of `EXC.EN`).

* Implementations **must recompute `EFF`** when writing `APPLY0=1` or `APPLY1=1`.

* After `APPLY0=1` or `APPLY1=1`, **implementations must latch** the effective `IE_MASK` (from `CAP.PREC.EXC.EN & ~SAE_effective`) into `CAP.PREC.STAT.IE_MASK` for debug and replay determinism.

#### 3.3.4 `CAP.PREC.STAT` — Effective State & Sticky Flags (RO)

|  Bits | Name               | Description                             |
| ----- | ------------------ | --------------------------------------- |
| 31:28 | **EFF_PET**        | Effective primary element type.         |
| 27:26 | **EFF_EW**         | Effective element width.                |
| 25:24 | **EFF_ACCW**       | Effective accumulator width.            |
|    23 | **EFF_ALT_EN**     | Alternate mode active.                  |
|    22 | **EFF_SAT**        | Effective saturation.                   |
| 21:19 | **EFF_FP_RMODE**   | Current rounding mode.                  |
|    18 | **EFF_Q**          | Quantized path enabled.                 |
| 17:16 | **EFF_PACK**       | Effective packing factor.               |
|    15 | **UNSUP_FMT**      | Unsupported ALT_FMT → fallback applied. |
|    14 | **DOWNCAST_TAKEN** | Safe down-cast performed in hardware.   |
|    13 | **SAT_HIT**        | Saturation event occurred.              |
|    12 | **FTZ_HIT**        | Subnormal flush-to-zero occurred.       |
|    11 | **EFF_ZMODE**      | Mirror effective defaults in use.       |
|    10 | **EFF_SAE**        | Mirror effective defaults in use.       |
|   9:8 | **EFF_NAN_POL**    | Mirror effective defaults in use.       |
|   7:4 | **IE_MASK**        | Latched view of `EXC.EN`.               |
|   3:0 | **RSV**            | Reserved (may include counters).        |

> **Reset defaults:** `PET=FP16`, `EW=16`, `ACCW=FP32`, `SAT=0`, `ALT_EN=0`.

#### 3.3.5 FP Exception Control — trap/SAE precedence

| CSR               | Addr  | Access | Description                                                                  |
| ----------------- | ----- | ------ | ---------------------------------------------------------------------------- |
| `CAP.PREC.EXC.EN` | 0x7D3 | RW     | FP exception enables: `NV,DZ,OF,UF,NX` (bits 4:0). Trap only if `SAE_DEF=0`. |
| `CAP.PREC.EXC.ST` | 0x7D4 | RO/W1C | Sticky flags `NV,DZ,OF,UF,NX` + `QNAN_SEEN,SNAN_SEEN`. Write 1 clears bit.   |

* **SAE precedence:** Per-instruction SAE overrides (e.g., via RSV `svon.fpctl`) take precedence over `CAP.PREC.MODE.SAE_DEF`.
* **Trap gate:** Traps for NV,DZ,OF,UF,NX are delivered **only if** the relevant bit is 1 in `EXC.EN` **and** the effective SAE is 0; otherwise stickies in `EXC.ST` are set and execution continues.
* **Stickies lifetime:** Stickies are **persistent** until W1C; they are **not** auto-cleared by `APPLY0/1`.

**Conformance:**

* Units **must** set sticky bits when exceptions/NaN observations occur.
* If `CAP.PREC.MODE.SAE_DEF=1` or a per-op SAE override is set, units **suppress** synchronous traps but still update stickies.

> **RSV note (optional but recommended):** you may add a lightweight one-shot prefix (e.g., `svon.fpctl`) to override **RC/SAE/ZMODE** for the *next* instruction without touching CAP. If absent, software can still adjust the defaults via `CAP.PREC.MODE` between ops.

### 3.4 Precision/Format Capability CSRs (RO)

* `CAP.PREC.ALU`: bits for `HAS_FP32`, `HAS_FP16`, `HAS_BF16`, `HAS_INT32/16/8`, `HAS_DOTPROD8` (if applicable), etc.
* `CAP.PREC.TEX`: support for `UNORM8/16`, `SNORM8/16`, `SRGB8_A8`, `FP16`, `FP32`, and optionally block-compressed families (BC1-7) as feature clusters.
* `CAP.PREC.IMG`: classes `R8/R16/R32_{U|S|F}`, `RG*`, `RGBA*`, typed-atomic flag(s).
* `CAP.PREC.RT`: payload/barycentrics: `PAYLOAD_FP32`, `PAYLOAD_FP16`, `BARY_FP32`, `BARY_FP16`, etc.

> These CSRs **do not change behavior**; they are for feature discovery and tuning.


### 3.5 Tiers & Hints (RO)

* `CAP.TIER.REG`: `{VGPR_TIER, SGPR_TIER}` (S/M/L/XL).
* `CAP.TIER.WAVE`: waves per CU (encoded ranges).
* `CAP.TIER.LDS`, `CAP.TIER.SAMP`, `CAP.TIER.RT`: relative tiers.
* `CAP.HINT.ALIGN`: bitmask of preferred alignments (16/32/64/128).
* `CAP.HINT.BVHGR`: preferred `RAY.NODE.PREFETCH` granularity (e.g., encoded {64,128,256}).
* `CAP.HINT.LDSBK`: LDS bank hint.

---

## 4. Conformance & reset defaults

* **Architectural:** All `XPHMG_*` domains (RSV/MTX/GFX/RT/XMEM) MUST observe `CAP.PREC.*` and `CAP.XMEM.*` state at the instruction boundary.
* **Advisory:** `CAP.FEAT*`, `CAP.PREC.{ALU,TEX,IMG,RT}`, `CAP.TIER.*`, `CAP.HINT.*` are **non-binding** (do not affect correctness).
* **Forward progress:** precision changes must not induce deadlock; units may stall without side effects until application completes.
* **Unsupported formats:** hardware **may** downgrade or ignore; it **must** set `UNSUP_FMT` in `CAP.PREC.STAT`.
* **Unsupported fields:** Read-as-zero / ignore-writes; set `CAP.PREC.STAT.UNSUP_FMT` only for precision/format issues, not for memory features.

* **Apply semantics:**
  * `CAP.PREC.MODE/APPLY0` and `CAP.PREC.ALT/APPLY1` are **write-one-to-apply** gates for the **PREC** group only.
  * **XMEM and SVMEM CSRs** take effect **at the next instruction boundary** with no extra APPLY bit; engines must observe the latest CAP state at decode (see XMEM §14 N-1).
  * Multi-CSR programming is allowed; the architecturally visible state is what is read **after** the final write. Implementations may internally stage changes as long as the boundary rule is respected.

* **Reset:**
  * `CAP.PREC.MODE`: `ZMODE=0` (*merge*), `SAE_DEF=0`, `NAN_POL=propagate`, other bits per your current defaults.
  * `CAP.XMEM.*`: all RW fields reset to 0 (meaning “implementation defaults”), except `ACTIVE_CLASS=0`.


---

## 5. Optional Runtime Profiles (RO)

If `CAP.FLAGS.HAS_PROFILES=1`:

| CSR            | Addr  | Access | Description                       |
| -------------- | ----- | ------ | --------------------------------- |
| `CAP.PROF.CUR` | 0x7C3 | RO     | Current profile (e.g., P0/P1/P2). |
| `CAP.PROF.SUP` | 0x7C4 | RO     | Bitmask of supported profiles.    |

> Profiles **do not** change architectural semantics; only **tiers/hints** may vary with the profile.

---

## 6. Programming Model

1. **Program base mode**

```asm
# Base: FP16 compute, accumulate FP32
li  t0, (0<<30)              # SAT=0
    | (0b000<<27)            # RNE
    | (0b10<<22)             # ACCW=FP32
    | (0b011<<19)            # PET=FP16
    | (0b01<<17)             # EW=16
    | (1<<31)                # APPLY0
csrw 0x7D0, t0               # CAP.PREC.MODE
```

2. **(Optional) Enable alternate FP8 mixed**

```asm
li  t1, (1<<30)              # ALT_EN=1
    | (0b0010<<26)           # ALT_FMT=FP8_E4M3
    | (0b01<<24)             # ALT_ACCW=FP16
    | (1<<22)                # MIXED=1
    | (1<<8)                 # ALT_SAT=1
    | (1<<31)                # APPLY1
csrw 0x7D1, t1               # CAP.PREC.ALT
```

3. **Inspect effective state**

```asm
csrr t2, 0x7D2               # CAP.PREC.STAT

# (Optional) Enable FP traps for overflow/invalid; keep others sticky-only
li  t3, ((1<<2) /*OF*/ | (1<<4) /*NV*/)
csrw 0x7D3, t3              # CAP.PREC.EXC.EN
```

4. **Dispatch workloads**
   Jobs submitted to `XPHMG_*` (GFX/ALU, MTX, RT) inherit the active `EFF` until the next application.

---

## 7. Rationale

* **A single “corner”**: discovery of capabilities + numerical policy in the same block facilitates tooling and drivers.
* **Architectural clarity**: separates what is **state (RW)** from what is **advisory (RO)**.
* **Evolvable**: `CAP.VERS` and the CSR grid allow for extension without breaking compatibility.
* **Portability**: formats/precisions are configurable, but capabilities remain optional and non-binding.

---

## 8. Glossary of the new/changed bits

* `ZMODE` — masked lane writeback policy (merge vs zeroing).
* `SAE_DEF` — default “Suppress All Exceptions” for FP ops.
* `NAN_POL` — global NaN policy (propagate / squash variants).
* `CAP.PREC.EXC.{EN,ST}` — FP exception enables/stickies.
* `CAP.XMEM.*` — architectural memory defaults and SVMEM for index-based gather/scatter, used by all domains.
* `IE_MASK` — latched, effective exception-enable view used during execution (mirrored in `CAP.PREC.STAT`).

### 8.1 Cross-references to other specs (editorial)

* Update **RSV** doc to state: predication zero/merge behavior is governed by `CAP.PREC.MODE.ZMODE`; per-op overrides are allowed if the optional `svon.fpctl` prefix is implemented.
* Update **XMEM** doc: its per-engine “programming” section should reference these CAP CSRs as the **architectural sources of truth**; engine-local mirrors (if any) are micro-architectural.
* Update **GFX/MTX/RT** docs: when describing memory behavior, point to `CAP.XMEM.DOM`, `…L1CFG`, `…CLS*`, and SVMEM controls for gather/scatter phases.


---
**License:** CC-BY-SA 4.0 — Open Specification maintained by Pedro H. M. Garcia.
