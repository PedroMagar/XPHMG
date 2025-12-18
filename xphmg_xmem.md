# RISC-V Vendor Extension **XPHMG_XMEM**
**Unified LDS / Cache / Streaming Memory Control**

**Category:** Vendor Extension (part of `XPHMG_*` family)  
**Version:** 0.1.0
**Author:** Pedro H. M. Garcia  
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview (Informative)

`XPHMG_XMEM` defines the explicit memory control model shared by all XPHMG execution domains. It provides a single architectural contract for LDS allocation, cache/streaming controls, coherence domains, memory classes, and indexed gather/scatter (SVMEM), with configuration sourced from CAP-owned state.

### 1.1 Conceptual Model

- XMEM is a control plane only: it does not change instruction semantics defined in other extensions; it supplies the configuration (classes, domains, streaming profiles, compression defaults, gather/scatter parameters) consumed by those instructions.
- Configuration lives in CAP (`CAP.XMEM.*`) and may be mirrored per-engine (XMIR.*) for performance; CAP is the sole architectural source of truth.
- Memory behavior is explicit and descriptor-driven: instructions and descriptors consume CAP defaults unless a non-zero override is provided.
- The model is domain-agnostic: RSV, MTX, RT, GFX, ACCEL, and CPU observe the same memory classes, domains, tiers, and SVMEM rules.

### 1.2 Architectural Guarantees and Invariants

- Determinism at instruction boundaries: an instruction observes latched CAP state; mid-instruction CSR writes do not affect it.
- Class and domain uniformity: a given class/domain identifier has the same meaning across all domains; there are no per-engine reinterpretations.
- Fallback to zero/ignore: if a feature bit in `CAP.XMEM.CAP` is clear, dependent controls are architecturally disabled and read as zero; writes are ignored safely.
- Descriptor override rules: descriptor fields override CAP defaults only when non-zero and supported; otherwise CAP defaults apply.
- Cross-domain visibility follows CAP-selected domains and scopes; fences order with respect to those domains.

### 1.3 Interaction with Other XPHMG Extensions

- CAP: defines the authoritative XMEM CSRs, feature bits, defaults, and visibility scopes. XMEM instructions and descriptors consume this state.
- RSV/MTX/RT/GFX/ACCEL: reuse XMEM for LDS allocation, cache/streaming hints, SVMEM gather/scatter, and class/domain selection; no extension redefines XMEM fields.
- PREC: provides element width and predication context for SVMEM (e.g., `EFF_EW`, `ZMODE`); XMEM gather/scatter honors these PREC controls.
- XDL: layout metadata referenced by XMEM regions is shared across engines; XMEM tiers (T0/T1/...) describe placement/latency, not layout.

### 1.4 Undefined or Reserved Behavior

- If an implementation does not support a documented feature bit in `CAP.XMEM.CAP`, related fields are reserved/RAZ/WI and have no architectural effect.
- Cross-engine register forwarding is not architecturally defined; all cross-engine dataflow must occur through XMEM-managed memory.
- NOTE: This behavior is intentionally left unspecified in v0.1 and may be clarified in a future revision for implementation-defined mirror modes beyond aliased or read-only.

### 1.5 Notes for Implementers (Informative)

- Keep CAP as the single programmed interface; mirrors are optional accelerators and must be coherent at instruction boundaries.
- Document which optional fields/features in `CAP.XMEM.CAP` are implemented; leave unimplemented fields RAZ/WI.
- When providing toolchain headers or firmware abstractions, surface CAP defaults and descriptor override precedence explicitly to avoid hidden policy.

---

## 2. Conventions & Terminology (Informative)

This section defines the vocabulary and notation used throughout XPHMG_XMEM. Terms are aligned with the RISC-V Vector specification style and reused across all XPHMG extensions.

### 2.1 Domains and Scopes

- **Domain (`dom`)**: coherence/visibility grouping selected from `CAP.XMEM.DOM` or descriptor override. Identifiers are shared across engines (e.g., `DOM_CPU`, `DOM_RT`, `DOM_MTX`, `DOM_ALL`).
- **Scopes**: ordering scopes follow the hierarchy `THREAD < WARP < WG < CLUSTER < SYSTEM`. Fence scopes in XMEM and LDS instructions refer to this hierarchy.

### 2.2 Classes and Partitioning

- **Memory Class (`mclass`)**: selects a class index (CL0..CL3) mapping to cache ways/QoS as defined by `CAP.XMEM.CLSMAP`. Classes are global; engines must not reinterpret class IDs.
- **Active Class**: the effective class used when `CLASS_PART=1`, taken from descriptor `mclass` if non-zero, else `CAP.XMEM.L1CFG.ACTIVE_CLASS`.
- **Partitioning**: `XMEM.CCTL.PARTITION` (or class mapping when `CLASS_PART=1`) defines how classes map to ways/regions; defaults live in CAP.XMEM.

### 2.3 Tiers

- **Tier (`tier`)**: placement/latency hint encoded in `XPHMG_MEMCTL`. Tiers describe expected residency, not layout:
  - `T0`: Tier-0 / shared hot data (quasi-register)
  - `T1`: LDS/XMEM working sets
  - `T2`: Coherent cache hierarchy residency
  - `T3`: DRAM / high-latency memory
- Tier semantics apply uniformly across domains; only `tier` and `mclass` determine placement hints.

### 2.4 SVMEM Terminology

- **SVMEM**: indexed gather/scatter model configured by `CAP.XMEM.SVMBASE/SVMIDX/SVMSCL/SVMLEN/SVMMODE`.
- **FOF (fault-only-first)**: controlled by `SVMSCL.FOF`; on first faulting element, write index to `RSV.SVFAULTI` and abort.
- **ZMODE**: masked-lane writeback policy from `CAP.PREC.MODE.ZMODE`, applied to gathers/scatters.

### 2.5 Configuration Sources

- **CAP.XMEM**: architectural source of truth for memory classes, domains, streaming profiles, compression defaults, descriptor policy, and SVMEM configuration.
- **XMIR.* (mirrors)**: optional vendor mirrors that must match CAP at instruction boundaries.
- **Descriptor (`XPHMG_MEMCTL`)**: per-job overrides for class, domain, streaming profile, security tag, and LDS hints; non-zero fields override CAP defaults.

### 2.6 Notation and Formatting

- Register/CSR names follow `CAP.XMEM.*` and `XMIR.*`.
- Hex and bit ranges use standard RISC-V notation (e.g., `bits[7:0]`).
- "RAZ/WI" indicates read-as-zero / write-ignored when a feature is disabled or unimplemented.

### 2.7 Notes for Implementers (Informative)

- When presenting APIs or headers, keep the shared definitions of `dom`, `mclass`, scopes, and tiers consistent across engines.
- Document which `CAP.XMEM.CAP` feature bits are implemented; treat unimplemented ones as RAZ/WI.
- TODO: Clarify interaction with any future domain/profile extensions if additional domain IDs or class ranges are added in later revisions.

---

## 3. Architectural Memory Model

This section describes the architectural view of memory levels, placement tiers, and visibility rules shared by all XPHMG engines. Presence of specific levels or features is advertised via `CAP.XMEM.CAP`; absent features are RAZ/WI.

### 3.1 Memory Hierarchy

Implementations **may** expose any subset of:

| Level            | Description                                                                                                    |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| **L0-LDS**       | Local Data Store (a.k.a. SLM). Banked SRAM per cluster, dynamically allocated via `XLDS.ALLOC`.                |
| **L1$-Cluster**  | Unified set-associative cache (tensor, texture, or byte data) with optional way-partitioning and line locking. |
| **L2$-Fabric**   | Shared cache across clusters. Optionally coherent with CPU.                                                    |
| **XDMA Engines** | Streaming, non-temporal DMA with optional inline decompression/packing.                                        |
| **UMF Fabric**   | Interconnect providing domain tags (`CPU`, `RT`, `MTX`, `ALL`).                                                |

**Coherence scopes:**  
`THREAD < WARP < WG < CLUSTER < SYSTEM`

All fences and LDS barriers honor these scopes.

Notes (informative):
- The scope hierarchy is the only architecturally observed ordering; implementations may add finer-grained internal levels, but ordering visible to software follows these scopes.
- If a level is absent per `CAP.XMEM.CAP`, related controls are RAZ/WI; software must not assume the presence of a specific cache/fabric level without checking capability bits.

### 3.2 Tier Model

Tiers are placement/latency hints carried in `XPHMG_MEMCTL.tier`. Tiers do not alter memory layout; they describe expected residency and access latency visible to all engines.

#### 3.2.1 Unified XMEM View

All engines (RSV, MTX, GFX, RT, ACCEL) view XMEM through the same descriptor format `XPHMG_MEMCTL`.  
Direct engine-to-engine register forwarding is not architecturally defined; all cross-engine dataflow MUST occur through XMEM regions.

#### 3.2.2 Memory Class for Hot Shared Data (`CLS_SHARED`)

Implementations MUST reserve at least one memory class ID for shared hot data:

- **`CLS_SHARED`** - identifies XMEM regions intended for cross-engine communication using Tier-0.

Other memory classes remain implementation-defined (e.g., `CLS_GBUF`, `CLS_MTX`, `CLS_RT`, etc.).

#### 3.2.3 Tier Field (`tier`) in `XPHMG_MEMCTL`

`XPHMG_MEMCTL` SHALL include a `tier` field (min. 2 bits) that indicates the intended tier of an allocated region:

| tier value | Meaning |
|-----------|---------|
| `T0` | Tier-0 / shared hot data (quasi-register) |
| `T1` | Regular LDS/XMEM |
| `T2` | Coherent memory / higher cache tiers |
| `T3` | DRAM / external memory |

A region marked as `CLS_SHARED / T0` is eligible for cross-engine handoff.

#### 3.2.4 Logical Ownership (`dom`)

`dom` indicates which engine currently owns or has priority access to the region:

- `DOM_MTX`, `DOM_RT`, `DOM_GFX`, `DOM_RSV`, `DOM_ACCEL`,  
- `DOM_CPU`,  
- or `DOM_ALL` for shared read access.

Changing `dom` on a `CLS_SHARED / T0` region MUST NOT imply physical data movement.  
It is strictly a logical ownership transition used to orchestrate pipeline stages (e.g., MTX->RT->GFX).
Unless overridden by descriptor, the default `dom` is sourced from `CAP.XMEM.DOM`; fence scopes in Section 7 apply to visibility between domains.

#### 3.2.5 Partitioning and Tier-0 Mapping (`XMEM.CCTL.PARTITION`)

`XMEM.CCTL.PARTITION` MUST describe which physical subrange(s) of LDS/XMEM are reserved for Tier-0 use.  
Only regions whose `tier` field equals `T0` MAY map into this Tier-0 partition.

Tier-0 regions:

- MUST provide lower-latency access than Tier-1,  
- MAY have size limitations,  
- MAY restrict aliasing,  
- MAY implement additional hazard or fencing semantics.

#### 3.2.6 Handoff Semantics

A handoff between engines (e.g., MTX->RT) is architecturally defined as:

```

1. Producer writes to region R marked as:
   R = CLS_SHARED / T0 / DOM_PRODUCER
2. Upon completion, software or micro-scheduler updates:
   R.dom = DOM_CONSUMER
3. Consumer reads region R without further data movement.

```

No explicit synchronization barriers are implied beyond normal XMEM ordering rules, unless otherwise specified by the engine's ISA extension.

#### 3.2.7 Engine Behavior

- Engines MUST treat any `CLS_SHARED / T0` region with matching `dom` as a valid hot data source or sink.  
- Engines MUST NOT read or write Tier-0 regions whose `dom` does not authorize access.  
- Engines MAY apply optimizations (prefetch, low-latency paths, co-issue) specifically for Tier-0.

#### 3.2.8 Fallback to Tier-1

If Tier-0 capacity is insufficient, engines MAY allocate regions as:

- `CLS_SHARED / T1 / DOM_*`

Such regions still participate in the unified dataflow model, but may incur additional latency and may require explicit synchronization barriers.

#### 3.2.9 Tier-1 Regions and Working Sets

Tier-1 (T1) regions are XMEM allocations intended to hold intermediate or reusable working sets that:

- Are larger than what can reasonably fit in Tier-0,
- Are still latency-sensitive enough to warrant LDS/XMEM placement,
- May be produced and consumed across multiple phases of the same engine or across different engines.

A Tier-1 region is any `XPHMG_MEMCTL`-described region whose `tier` field equals `T1`, regardless of `mclass` or `dom`.

Typical uses include (non-exhaustive):

- MTX tile staging and pre/post-processing buffers.
- RT BVH node construction, compaction, and scratch storage.
- GFX G-buffer staging, temporal history, and intermediate surfaces.
- ACCEL tensor blocking and intermediate activation buffers.
- RSV/CPU scratch data for loop transforms, gather/scatter staging, and reduction buffers.

##### 3.2.9.1 Access Semantics

- Tier-1 regions follow standard XMEM ordering rules.
- Engines MAY freely allocate, read and write Tier-1 regions, subject to `dom` permissions.
- Unlike Tier-0, Tier-1 **does not** imply special hot-handoff semantics. Changing `dom` on a Tier-1 region does **not** carry the same implicit "handoff" meaning; it is a policy hint for arbitration and scheduling.

##### 3.2.9.2 Relationship to Tier-0 and Tier-2+

- Tier-1 sits between Tier-0 and Tier-2:

  - Tier-0: shared hot data (`CLS_SHARED / T0`) used as quasi-register for cross-engine handoff.
  - Tier-1: larger working sets, with LDS/XMEM latency and capacity.
  - Tier-2+: coherent caches / DRAM, not directly controlled by XPHMG.

- A typical pattern is:

  1. Data is constructed, re-tiled, or transformed in Tier-1.
  2. A subset is promoted or projected into Tier-0 (`CLS_SHARED / T0`) for cross-engine consumption.
  3. Results may be written back to Tier-1 or to higher tiers for long-term storage.

##### 3.2.9.3 Partitioning and QoS

`XMEM.CCTL.PARTITION` MAY also expose how Tier-1 capacity is partitioned across memory classes and engines, e.g.:

- Preferred size per `mclass` (e.g., `CLS_GBUF`, `CLS_RT`, `CLS_MTX`).
- Engine-local vs. shared Tier-1 pools.
- Optional QoS hints (priority levels) for Tier-1 allocations.

These details are implementation-defined but MUST NOT change the architectural semantics: a region marked as `tier = T1` MUST preserve LDS/XMEM-like latency and be accessible by any engine that holds appropriate `dom` permissions.

#### 3.2.10 XDL Tiles in Tier-0 and Tier-1

When a region is marked as `CLS_SHARED / T0`, and its layout metadata corresponds to `XDL_TILE4x4_ROWMAJOR` or a composite tile, engines MUST interpret the memory content as an XDL tile suitable for cross-engine consumption.

- Tier-0 regions are intended for **hot tiles** produced by MTX and consumed by RSV/GFX/RT.  
- Tier-1 regions are the recommended location for constructing and linking composite tiles.

No engine is allowed to reinterpret the tile using a non-XDL layout when the region declares an XDL tile layout.

#### 3.2.11 Tier-2 and Tier-3 Regions

While Tier-0 and Tier-1 are explicitly modeled as LDS/XMEM-backed regions, Tier-2 and Tier-3 represent how such regions are serviced by the wider memory system:

- **Tier-2 (T2)** - Regions whose primary residency is expected to be within the coherent cache hierarchy (L1/L2/fabric-level shared memory).
- **Tier-3 (T3)** - Regions whose primary residency is in external DRAM or equivalent high-latency memory.

A region with `tier = T2`:

- Is allocated in the same virtual/physical address space as other regions.
- Is expected to benefit from cache locality and reuse across engines and CPU, subject to `mclass` and `dom`.
- Uses normal cacheable XMEM operations (e.g., `XMEM.LD/ST/PREF`), honoring coherence scopes and fences.

A region with `tier = T3`:

- Is expected to be accessed primarily via streaming or non-temporal paths (e.g., XDMA engines, `XMEM.*` with non-temporal or streaming profiles).
- May bypass or minimally use caches, depending on `mclass` and streaming profile hints.
- Is still part of the coherent address space unless explicitly declared non-coherent by implementation-specific policy.

Semantics:

- T2 vs T3 do **not** affect architectural correctness; they are performance hints attached to `XPHMG_MEMCTL`.
- Engines MUST treat loads/stores to T2/T3 regions as globally visible and coherent, subject to the same ordering rules as CPU memory.
- Implementations MAY map T2/T3 to one or more of the memory levels described in Section 3.1 (L1$/L2$/XDMA/UMF), but no particular mapping is mandated.
NOTE: If coherence fabric is absent for a level, implementations must still honor architectural ordering scopes; performance effects are implementation-defined.

### 3.3 Addressing & Visibility Rules

- Default `dom` and `mclass` are taken from `CAP.XMEM.DOM` and `CAP.XMEM.CLSMAP`; non-zero descriptor fields override these defaults.
- XMEM and LDS fences order effects according to the scope hierarchy in Section 3.1 and the current `dom`. Mid-instruction CSR writes do not affect in-flight operations.
- Gather/scatter uses the effective element width from `CAP.PREC.STAT.EFF_EW` when `SVMLEN=0`; masked lanes follow `CAP.PREC.MODE.ZMODE`. Bounds checking, when enabled by `SVMMODE`, is applied per-element.
- Streaming and compression hints flow from `CAP.XMEM.STREAM_CTL/SPROF*` and `CAP.XMEM.COMP_CTL` unless a descriptor provides a non-zero override; hints cannot change architectural correctness.
- Virtual/physical addressing follows the host ISA; XMEM introduces no additional address translation rules. Domains and classes affect visibility/placement, not address spaces.

### 3.4 Notes for Implementers (Informative)

- Treat tiers strictly as performance guidance; correctness must not depend on any specific mapping of tiers to physical levels.
- Keep domain/class identifiers uniform across engines; avoid engine-local reinterpretations in firmware or drivers.
- When a capability bit is clear in `CAP.XMEM.CAP`, expose related CSRs as RAZ/WI and keep mirrors coherent with that behavior.

---

## 4. Architectural State

This section defines the architectural state that governs XMEM. CAP-owned CSRs are authoritative; mirrors are optional cached views that must not diverge at instruction boundaries.

### 4.1 CAP.XMEM CSRs

#### 4.1.1 Architectural Defaults and Effective State

Unless overridden by a descriptor or instruction immediate, engines must use the following `CAP.XMEM.*` state:

- `CAP.XMEM.L1CFG`, `CAP.XMEM.L2CFG` - cache line size, replacement, locking, active class
- `CAP.XMEM.CLSMAP`, `CAP.XMEM.LDS_BUDGET` - class/way mapping and LDS soft budgets
- `CAP.XMEM.DOM` - default coherence domain
- `CAP.XMEM.STREAM_CTL`, `CAP.XMEM.SPROF[0..3]` - streaming detector/profiles
- `CAP.XMEM.COMP_CTL` - inline-compression defaults
- `CAP.XMEM.DESCPOL` - descriptor fallback policies
- `CAP.XMEM.SVMBASE/SVMIDX/SVMSCL/SVMLEN/SVMMODE` - gather/scatter control
- `CAP.XMEM.EVENTS` - sticky status (W1C)

Notes and invariants:
- If a feature bit in `CAP.XMEM.CAP` = 0, the corresponding behavior is architecturally disabled; the register reads as zero and ignores writes (RAZ/WI).
- Descriptor fields (`mclass`, `sprof_id`, `dom`, `sec`) override CAP defaults only when non-zero and supported; otherwise CAP defaults apply.
- Effective state is latched at instruction boundaries; mid-instruction CSR writes do not affect in-flight operations.

### 4.2 Optional Engine-Local Mirrors (XMIR.*) and synchronization rules

These CSRs are optional vendor mirrors used for fast reprogramming or debug. They must reflect the canonical architectural state stored in `CAP.XMEM.*` at every instruction boundary. Software must consider `CAP.XMEM.*` as the single source of truth; `XMIR.*` are implementation conveniences only.

**N-0 (Mirror Coherence).** Immediately before any memory instruction issues, the mirror must equal the corresponding `CAP.XMEM.*` value. If the platform supports deferred apply, mirrors must synchronize before the next instruction with side effects.

Vendors may implement non-architectural mirror CSRs inside XMEM (e.g., `xmem_l1cfg_mirror`) for fast reprogramming. At the boundary before issuing a memory op, the mirror must equal CAP state, or the engine must latch CAP into the mirror. If the platform supports deferred apply, mirrors must be synchronized on the next instruction boundary before any memory side-effect.

#### 4.2.1  XMIR.LDSZ (RO)

Mirror of `CAP.XMEM.PARAMS.LDS_KiB`. Reports total LDS bytes per cluster.

#### 4.2.2  XMIR.LDSPART (RW)

Snapshot of active LDS allocations. Read-only if hardware auto-partitions.

#### 4.2.3  XMIR.L1CFG (RW)  - Mirror of `CAP.XMEM.L1CFG`

|  Bits | Field        | Description                                       |
| :---: | :----------- | :------------------------------------------------ |
|  7:0  | `WAYS`       | Associativity (hint if fixed)                     |
|  15:8 | `LINE_SZ`    | Sector size (0=32 B, 1=64 B, 2=128 B, 3=impl.)    |
| 19:16 | `REPL`       | Replacement (0 LRU - 1 PLRU - 2 LFU - 3 RR/impl.) |
|   20  | `LOCK_EN`    | Enable line/way lock ops                          |
| 31:24 | `ACTIVE_CLS` | Active class index (0-3)                          |

#### 4.2.4  XMIR.CLSMAP (RW)  - Mirror of `CAP.XMEM.CLSMAP`

| Bits  | Field         | Description      |
| ----- | ------------- | ---------------- |
| 7:0   | `CL0_WAYMASK` | L1$ ways for CL0 |
| 15:8  | `CL1_WAYMASK` | L1$ ways for CL1 |
| 23:16 | `CL2_WAYMASK` | L1$ ways for CL2 |
| 31:24 | `CL3_WAYMASK` | L1$ ways for CL3 |
| 35:32 | `CL0_QOS`     | QoS weight       |
| 39:36 | `CL1_QOS`     | QoS weight       |
| 43:40 | `CL2_QOS`     | QoS weight       |
| 47:44 | `CL3_QOS`     | QoS weight       |

*Default mapping recommendation:*  CL0 = RT - CL1 = MTX - CL2 = HYB - CL3 = OTHER.

#### 4.2.5  XMIR.LDS_BUDGET (RW)  - Mirror of `CAP.XMEM.LDS_BUDGET`

| Bits | Field | Description |
| 15:0 | `CL0_KiB` | Budget for CL0 |
| 31:16 | `CL1_KiB` | Budget for CL1 |
| 47:32 | `CL2_KiB` | Budget for CL2 |
| 63:48 | `CL3_KiB` | Budget for CL3 |

Overflow -> `CAP.XMEM.EVENTS.BUDGET_OVF`.

#### 4.2.6  XMIR.L2CFG (RW)  - Mirror of `CAP.XMEM.L2CFG`

Victim policy, QoS, snoop coherence, and compression threshold.

#### 4.2.7  XMIR.DOM (RW)  - Mirror of `CAP.XMEM.DOM`

Default domain: `1=CPU  2=RT  3=MTX  15=ALL`.

#### 4.2.8  XMIR.STREAM_CTL (RW)  - Mirror of `CAP.XMEM.STREAM_CTL`

Fields: `stride`, `pf_depth`, `nontemp_en`.

#### 4.2.9  XMIR.SPROF[0-3] (RW)  - Mirror of `CAP.XMEM.SPROF[0-3]`

Each profile: `stride[15:0]`, `pf_depth[23:16]`, `nt_policy[27:24]`.

#### 4.2.10  XMIR.COMP_CTL (RW)  - Mirror of `CAP.XMEM.COMP_CTL`

Compression mode/level/dictionary for inline compression.

#### 4.2.11  XMIR.DESCPOL (RW)  - Mirror of `CAP.XMEM.DESCPOL`

| Bits | Field | Description |
| 1:0 | `DEF_CLASS` | Default class for jobs without `mclass` |
| 2 | `HYB_ENABLE` | Allow HYB (CL2) borrowing |
| 5:3 | `HYB_BORROW_QOS` | Borrow priority |
| 6 | `STRICT_DOM` | Reject domain mismatch |
| 7 | `RSV` | Reserved |

#### 4.2.12  XMIR.EVENTS (W1C Mirror of `CAP.XMEM.EVENTS`)

Sticky flags: `INV_MISS`, `LOCK_OVF`, `LDS_OOM`, `DMA_ERR`, `COMP_ERR`, `BUDGET_OVF`, `SEC_VIOL`, `HYB_FALLBK`.

### 4.3 Descriptor `XPHMG_MEMCTL`

#### C structure
```c
struct XPHMG_MEMCTL {
  uint8_t  dom;         // DOM_CPU=1, DOM_RT=2, DOM_MTX=3, DOM_ALL=15
  uint8_t  l1_hint;     // {LOCK,STREAM,NONTEMP} | waymask bits
  uint8_t  l2_hint;     // {PREFONLY,BYPASS,COMP_EN} | QoS bits
  uint8_t  lds_hint;    // {ALLOC_LOG2,BANK_SWZ,MCAST}
  uint16_t line_sz;     // 32/64/128
  uint16_t qos;         // Priority or stall budget
  uint64_t lds_softpin; // Best-effort LDS bytes to pin

  // Additional behavior
  uint8_t  mclass;      // Memory class index 0-3 (maps to xmem_clsmap)
  uint8_t  sprof_id;    // Streaming profile id 0-3
  uint8_t  hyb;         // Hybrid/shared job (1=HYB)
  uint8_t  sec;         // Security tag if SEC_DOM=1
  uint32_t rsv0;        // Reserved
};
```

#### Semantics

- `mclass` maps to one of the classes CL0-CL3.
- `sprof_id` selects streaming profile (0-3).
- `hyb=1` indicates hybrid job (prefer CL2).
- `sec` ignored unless `SEC_DOM=1`.
- If unsupported fields are set, hardware ignores them safely.

### 4.2 Optional Engine-Local Mirrors (XMIR.*) and synchronization rules

These CSRs are **optional vendor mirrors** used for fast reprogramming or debug.
They **MUST** reflect the canonical architectural state stored in `CAP.XMEM.*` at every instruction boundary.
Software must consider **`CAP.XMEM.*` as the single source of truth**; the `XMIR.*` registers are implementation conveniences only.

**N-0 (Mirror Coherence).**
Immediately before any memory instruction issues, the mirror **must equal** the corresponding `CAP.XMEM.*` value.
If the platform supports *deferred apply*, mirrors must synchronize before the next instruction with side effects.

Vendors may implement non-architectural *mirror CSRs* inside XMEM (e.g., `xmem_l1cfg_mirror`) for fast reprogramming.
**Rule:** at the boundary before issuing a memory op, the mirror **must equal** CAP state, or the engine must **latch CAP** into the mirror. If the platform supports *deferred apply*, mirrors must be synchronized on the next instruction boundary before any memory side-effect.

#### 4.2.1  XMIR.LDSZ (RO)

Mirror of `CAP.XMEM.PARAMS.LDS_KiB`.
Reports total LDS bytes per cluster.

#### 4.2.2  XMIR.LDSPART (RW)

Snapshot of active LDS allocations.
Read-only if hardware auto-partitions.

#### 4.2.3  XMIR.L1CFG (RW)  - Mirror of `CAP.XMEM.L1CFG`

|  Bits | Field        | Description                                       |
| :---: | :----------- | :------------------------------------------------ |
|  7:0  | `WAYS`       | Associativity (hint if fixed)                     |
|  15:8 | `LINE_SZ`    | Sector size (0=32 B, 1=64 B, 2=128 B, 3=impl.)    |
| 19:16 | `REPL`       | Replacement (0 LRU - 1 PLRU - 2 LFU - 3 RR/impl.) |
|   20  | `LOCK_EN`    | Enable line/way lock ops                          |
| 31:24 | `ACTIVE_CLS` | Active class index (0-3)                          |

#### 4.2.4  XMIR.CLSMAP (RW)  - Mirror of `CAP.XMEM.CLSMAP`

| Bits  | Field         | Description      |
| ----- | ------------- | ---------------- |
| 7:0   | `CL0_WAYMASK` | L1$ ways for CL0 |
| 15:8  | `CL1_WAYMASK` | L1$ ways for CL1 |
| 23:16 | `CL2_WAYMASK` | L1$ ways for CL2 |
| 31:24 | `CL3_WAYMASK` | L1$ ways for CL3 |
| 35:32 | `CL0_QOS`     | QoS weight       |
| 39:36 | `CL1_QOS`     | QoS weight       |
| 43:40 | `CL2_QOS`     | QoS weight       |
| 47:44 | `CL3_QOS`     | QoS weight       |

*Default mapping recommendation:*  CL0 = RT - CL1 = MTX - CL2 = HYB - CL3 = OTHER.

#### 4.2.5  XMIR.LDS_BUDGET (RW)  - Mirror of `CAP.XMEM.LDS_BUDGET`

| Bits | Field | Description |
| 15:0 | `CL0_KiB` | Budget for CL0 |
| 31:16 | `CL1_KiB` | Budget for CL1 |
| 47:32 | `CL2_KiB` | Budget for CL2 |
| 63:48 | `CL3_KiB` | Budget for CL3 |

Overflow -> `CAP.XMEM.EVENTS.BUDGET_OVF`.

#### 4.2.6  XMIR.L2CFG (RW)  - Mirror of `CAP.XMEM.L2CFG`

Victim policy, QoS, snoop coherence, and compression threshold.

#### 4.2.7  XMIR.DOM (RW)  - Mirror of `CAP.XMEM.DOM`

Default domain: `1=CPU  2=RT  3=MTX  15=ALL`.

#### 4.2.8  XMIR.STREAM_CTL (RW)  - Mirror of `CAP.XMEM.STREAM_CTL`

Fields: `stride`, `pf_depth`, `nontemp_en`.

#### 4.2.9  XMIR.SPROF[0-3] (RW)  - Mirror of `CAP.XMEM.SPROF[0-3]`

Each profile: `stride[15:0]`, `pf_depth[23:16]`, `nt_policy[27:24]`.

#### 4.2.10  XMIR.COMP_CTL (RW)  - Mirror of `CAP.XMEM.COMP_CTL`

Compression mode/level/dictionary for inline compression.

#### 4.2.11  XMIR.DESCPOL (RW)  - Mirror of `CAP.XMEM.DESCPOL`

| Bits | Field | Description |
| 1:0 | `DEF_CLASS` | Default class for jobs without `mclass` |
| 2 | `HYB_ENABLE` | Allow HYB (CL2) borrowing |
| 5:3 | `HYB_BORROW_QOS` | Borrow priority |
| 6 | `STRICT_DOM` | Reject domain mismatch |
| 7 | `RSV` | Reserved |

#### 4.2.12  XMIR.EVENTS (W1C Mirror of `CAP.XMEM.EVENTS`)

Sticky flags: `INV_MISS`, `LOCK_OVF`, `LDS_OOM`, `DMA_ERR`, `COMP_ERR`, `BUDGET_OVF`, `SEC_VIOL`, `HYB_FALLBK`.

### 4.3 Descriptor `XPHMG_MEMCTL`

#### C structure
```c
struct XPHMG_MEMCTL {
  uint8_t  dom;         // DOM_CPU=1, DOM_RT=2, DOM_MTX=3, DOM_ALL=15
  uint8_t  l1_hint;     // {LOCK,STREAM,NONTEMP} | waymask bits
  uint8_t  l2_hint;     // {PREFONLY,BYPASS,COMP_EN} | QoS bits
  uint8_t  lds_hint;    // {ALLOC_LOG2,BANK_SWZ,MCAST}
  uint16_t line_sz;     // 32/64/128
  uint16_t qos;         // Priority or stall budget
  uint64_t lds_softpin; // Best-effort LDS bytes to pin

  // Additional behavior
  uint8_t  mclass;      // Memory class index 0-3 (maps to xmem_clsmap)
  uint8_t  sprof_id;    // Streaming profile id 0-3
  uint8_t  hyb;         // Hybrid/shared job (1=HYB)
  uint8_t  sec;         // Security tag if SEC_DOM=1
  uint32_t rsv0;        // Reserved
};
```

#### Semantics

* `mclass` maps to one of the classes CL0-CL3.
* `sprof_id` selects streaming profile (0-3).
* `hyb=1` indicates hybrid job (prefer CL2).
* `sec` ignored unless `SEC_DOM=1`.
* If unsupported fields are set, hardware ignores them safely.

---

## 5. Execution Model

This section describes how XMEM-controlled behaviors are applied during execution, focusing on SVMEM gather/scatter, LDS usage, and cache/streaming interaction. It builds on CAP-provided state and the scope/dom/class model defined in Sections 2-4.

### 5.1 SVMEM Gather/Scatter Semantics

SVMEM defines indexed loads/stores governed by CAP and optional descriptor overrides. It is shared across RSV, MTX, RT, GFX, and any engine that issues indexed memory ops.

#### 5.1.1 Index addressing model

* **Source of control:** `CAP.XMEM.SVMBASE`, `...SVMIDX`, `...SVMSCL`, `...SVMLEN`, `...SVMMODE`.
* **Effective element size:** If `SVMLEN=0`, derive from `CAP.PREC.STAT.EFF_EW`; otherwise use `SVMLEN`.
* **Scale & sign:** `SVMSCL.SCALE  in  {1,2,4,8,16}`, `SVMSCL.SIGNED` selects sign-extended indices.
* **Gather vs scatter:** `SVMSCL.GS` bit selects load (gather) or store (scatter).
* **FOF:** If `SVMSCL.FOF=1`, on the first faulting element *i*, store *i* into `RSV.SVFAULTI` and abort.

Additional rules:
- Bounds checking, when enabled by `SVMMODE.BOUND_CHECK`, applies per element. If `ALLOW_OOB_ZERO=1`, out-of-range elements are zeroed instead of faulting.
- When `SVMLEN=0`, element width tracks PREC state; software must ensure PREC and XMEM remain consistent at instruction boundaries.

#### 5.1.2 Predication and masked lanes

* Per-lane requests are enabled by the current RSV mask.
* **Masked lanes writeback policy**: obey `CAP.PREC.MODE.ZMODE` (zeroing vs merge). For gathers, zeroing writes zeros to destination; for scatters, masked lanes **do not** perform a store.

#### 5.1.3 Address computation

```
addr[i] = SVMBASE + sext_or_zext( IDX[i] ) * SCALE
```

OOB protection is implementation-defined; if `SVMMODE.BOUND_CHECK=1` and an address is out-of-range, treat as fault (or zero if `ALLOW_OOB_ZERO=1`).

*Ordering:* Each issued element follows the domain and scope rules selected from `CAP.XMEM.DOM` or descriptor overrides. Mid-instruction CSR changes do not affect in-flight elements.

#### 5.1.4 Memory ordering and domains

* Default domain comes from `CAP.XMEM.DOM` unless overridden by descriptor or instruction hint.
* Visibility is governed by fences (see Section 7). The engine must honor domain-scope fences.

Notes:
- SVMEM semantics are shared across engines; there are no engine-specific reinterpretations of `SVMSCL`, `SVMMODE`, or FOF behavior.
- Fault reporting is per instruction: fault-only-first writes the index of the first faulting element and terminates the operation; subsequent elements do not issue.

### 5.2 LDS Model

LDS is a software-managed store, with allocation, barriers, and budgets governed by CAP and optional descriptor hints.

- **Allocation/Free:** `XLDS.ALLOC`/`XLDS.FREE` behavior is unchanged; when `CLASS_PART=1`, allocation consumes from the caller's effective class (`mclass` override or `ACTIVE_CLASS`).
- **Budgets:** If `CAP.XMEM.LDS_BUDGET` is non-zero for a class, engines must apply best-effort enforcement; overflows set `CAP.XMEM.EVENTS.BUDGET_OVF`.
- **Barriers:** `XLDS.BAR` orders LDS operations up to the requested scope (Section 3.1). All engines must observe LDS barriers consistently.
- **Atomics:** `XLDS.ATOMS` follow standard atomic semantics within LDS; ordering follows the requested scope.
- **Swizzle/Copy:** `XLDS.CPY` may apply bidirectional swizzle if `CAP.XMEM.CAP.BIDIR_SWZ=1`; correctness must not depend on swizzle.

Notes:
- LDS allocations are not implicitly coherent with caches; visibility across domains requires appropriate barriers/fences per Section 7.
- Class partitioning (`CLASS_PART`) influences allocation source but not architectural correctness; failure modes are reported via EVENTS and/or zero return on alloc.

### 5.3 Cache/Streaming Behavior

Cache and streaming behavior are configured via CAP defaults with optional descriptor overrides. They influence placement and performance, not correctness.

- **Class mapping:** `CAP.XMEM.CLSMAP` (or `XMEM.CCTL PARTITION` when `CLASS_PART=1`) defines way/QoS mapping for classes; all engines must interpret class IDs uniformly.
- **Streaming profiles:** Defaults come from `CAP.XMEM.STREAM_CTL` and `CAP.XMEM.SPROF[0..3]`; descriptor `sprof_id` overrides when non-zero.
- **Compression defaults:** `CAP.XMEM.COMP_CTL` supplies default mode/level/dictionary; descriptor/compressed opcodes may override when supported.
- **Hints and visibility:** Hints such as `{LOCK, STREAM, NONTEMP, PREFONLY, BYPASS}` affect placement/prefetching but must not alter architecturally visible data or ordering.

Notes:
- If a feature is disabled in `CAP.XMEM.CAP` (e.g., compression or streaming), related controls are RAZ/WI; instructions must still behave correctly without that feature.
- Implementations may differ internally in how hints are realized; architectural results (data, ordering, fault behavior) must be identical.

---

## 6. Instruction Set Overview

This section summarizes instruction families that consume XMEM configuration. Semantics and correctness are governed by CAP state and descriptor overrides; hints affect placement/performance only.

- **Allocation/Free.** `XLDS.ALLOC`/`XLDS.FREE` are unchanged; when `CAP.XMEM.CAP.CLASS_PART=1`, allocation consumes from the caller's active class (descriptor `mclass` if non-zero, else `CAP.XMEM.L1CFG.ACTIVE_CLASS`).
- **Budgets.** If `CAP.XMEM.LDS_BUDGET` is non-zero for a class, engines apply best-effort enforcement; overflow sets `CAP.XMEM.EVENTS.BUDGET_OVF`.
- **Bank swizzle.** If `CAP.XMEM.CAP.BIDIR_SWZ=1`, `XLDS.CPY` may swizzle in both directions as a hint; correctness must not rely on swizzle.

### 6.1 LDS instructions (`XLDS.*`)

| Mnemonic                          | Summary                                                                         |
|-----------------------------------|---------------------------------------------------------------------------------|
| **XLDS.ALLOC rd, rs_size, flags** | Allocate LDS space; return base or 0 on failure.                                |
| **XLDS.FREE rs_base**             | Release allocated LDS region.                                                   |
| **XLDS.BAR imm_scope**            | Barrier ordering LDS ops up to given scope.                                     |
| **XLDS.ATOMS op, [addr], val**    | Atomic ops on LDS.                                                              |
| **XLDS.CPY rd, rs, len, flags**   | Copy between LDS and regs/global. If `BIDIR_SWZ=1`, swizzle in both directions. |

Notes:
- `XLDS.BAR` scopes map to Section 3.1; barriers order LDS effects relative to the specified scope and current domain.
- `XLDS.ALLOC` failure returns 0 without modifying state.
- `XLDS.ATOMS` follow standard atomicity within LDS; ordering follows the requested scope.

### 6.2 Cache & streaming (`XMEM.*`)

| Mnemonic                        | Summary                                                             |
|---------------------------------|---------------------------------------------------------------------|
| **XMEM.LDG rd, [addr], hint**   | Load with hint (L1_ONLY, L2_ONLY, LDS_PREF, NONTEMP, LOCK, STREAM). |
| **XMEM.STG [addr], rs, hint**   | Store with hint.                                                    |
| **XMEM.PREF [addr], len, hint** | Prefetch (L1/L2/STREAM). Non-faulting.                              |
| **XMEM.CCTL op, arg0, arg1**    | Cache control ops (INV, FLUSH, LOCK, UNLOCK, PARTITION).            |
| **XMEM.STREAM cfg0, cfg1**      | Program streaming detector or profile.                              |

**CCTL PARTITION behavior:**  
When `CLASS_PART=1`, accepts `class_id, waymask` to reprogram a class mapping (`xmem_clsmap`).  
Otherwise behaves as legacy range-based partition control.

Notes:
- Hints `{LOCK, STREAM, NONTEMP, PREFONLY, BYPASS}` influence placement/prefetch only; architectural data and ordering must not change.
- If a feature (e.g., streaming, compression) is disabled in `CAP.XMEM.CAP`, related hints are RAZ/WI and must not trap.

### 6.3 Swizzle operations (`XSWZ.*`)
| Mnemonic                                                | Summary                             |
|---------------------------------------------------------|-------------------------------------|
| **XSWZ.TEX2TEN dst, src, layout_tex, layout_ten, tile** | Texture -> tensor layout conversion. |
| **XSWZ.TEN2TEX dst, src, layout_ten, layout_tex, tile** | Tensor -> texture layout conversion. |

Notes:
- Layout identifiers are implementation-defined unless standardized; operations must preserve data values.
- Swizzle ops are treated as hints for layout conversion; correctness is invariant to internal realization.

### 6.4 Optional inline compression

| Mnemonic                               | Summary                |
|----------------------------------------|------------------------|
| **XMEM.DLOADC rd, [addr], len, mode**  | Load compressed data.  |
| **XMEM.DSTOREC [addr], rs, len, mode** | Store compressed data. |

### 6.5 Compression I/O alignment

* **Defaults** (`mode/level/dictionary`) come from `CAP.XMEM.COMP_CTL`.
* The `XMEM.DLOADC/DSTOREC` opcodes remain; if `CAP.XMEM.CAP.COMPRESS=0`, they must trap as *Illegal Instruction* or act as normal LD/ST (implementation choice, document it).

NOTE: If inline compression is unsupported by an engine, it must either trap as illegal or execute uncompressed while signaling `COMP_ERR` in `CAP.XMEM.EVENTS`, consistent with Section 11.

---

## 7. Ordering, Fences & Scopes

This section describes the ordering primitives and how scope/domain/class selections affect visibility across engines. Scopes follow Section 3.1; domains and classes come from CAP or descriptor overrides.

### 7.1 Scope and Domain Model

- Fence scopes use the hierarchy `THREAD < WARP < WG < CLUSTER < SYSTEM` (Section 3.1).
- Default domain is `CAP.XMEM.DOM` unless overridden by descriptor or instruction hint; class mapping is from `CAP.XMEM.CLSMAP` or descriptor `mclass`.
- Fences order effects with respect to the scope and domain in effect when the fence is executed; mid-instruction CSR writes do not retroactively affect in-flight operations.

### 7.2 Fence and Ordering Operations

- **Cross-domain ordering:** `XFENCE.PIPE {rt|mtx|all}` orders all writes below the specified domain to be visible to subsequent reads.
- **Event tokens:** `XFENCE.SIGNAL` / `XFENCE.WAIT` synchronize producers and consumers; visibility still follows scope/domain rules.
- **Cache/LDS visibility:** Producers writing LDS must execute `XLDS.BAR >=CLUSTER` or `XMEM.CCTL FLUSH` before signaling consumers reading via L1/L2.
- **Global fence:** if `ff=global`, completion includes DMA/streams whose policy came from current CAP defaults or a descriptor override.
- **CLASS_PART interaction:** When `CLASS_PART=1`, fences must honor the active class selection (descriptor `mclass` if non-zero, else `CAP.XMEM.L1CFG.ACTIVE_CLASS`) when ordering class-mapped resources.

### 7.3 Ordering Rules and Guarantees

- Fences and barriers order all memory operations issued by the engine up to the specified scope, including LDS ops, XMEM loads/stores, and cache control operations.
- Prefetch (`XMEM.PREF`) is non-faulting and does not provide visibility guarantees by itself; ordering requires an explicit fence/barrier.
- Streaming and DMA configured via CAP or descriptors are included in ordering only when a global fence is requested (`ff=global`).
- Locks/pins set via `XMEM.CCTL` (`LOCK/UNLOCK`) are subject to the same scope ordering; correctness must not depend on internal lock implementation details.

### 7.4 Notes for Implementers (Informative)

- Preserve the shared meaning of scopes across all engines; avoid engine-local reinterpretations of fence scopes.
- Ensure mirror CSRs (if present) are synchronized before fences issue, so ordering reflects effective CAP state.
- Document any implementation choices for `ff=global` coverage of DMA/stream engines; software should not assume coverage beyond what is stated.

---

## 8. Security & Isolation

This section describes optional security partitioning for XMEM-controlled resources. Security behavior is enabled by `xmem_cap.SEC_DOM`; when disabled, related fields are RAZ/WI.

### 8.1 Security Tagging

- When `SEC_DOM=1`, LDS allocations, cache ways, and DMA/stream contexts may carry a security tag (`sec`, recommended 2 bits):
  - `00 = public`
  - `01 = trusted`
  - `10 = secure`
  - `11 = reserved`
- Security tags may be supplied via CAP defaults or descriptor overrides; when absent, default is implementation-defined but must be documented.

### 8.2 Enforcement and Events

- Accesses violating the active security policy must raise `xmem_events.SEC_VIOL` (reported via `CAP.XMEM.EVENTS` / `XMIR.EVENTS`).
- Security enforcement applies consistently across LDS, cache classes, and DMA/stream engines that consume XMEM policy.
- If `SEC_DOM=0`, security tags are ignored (RAZ/WI); accesses must not fault based solely on `sec`.

### 8.3 Interaction with Domains and Classes

- Security tags are orthogonal to `dom` and `mclass`; an access must satisfy both domain/class rules and security policy.
- Partitioning (`CLASS_PART`) and tier selection do not alter security semantics; they influence placement only.

### 8.4 Notes for Implementers (Informative)

- Document supported `sec` encodings and any implementation-defined defaults.
- Ensure that mirrors (XMIR) reflect security-related state when present; violations must be reported uniformly via EVENTS.
- TODO: Clarify interaction with other XPHMG security mechanisms if additional domains or privilege models are introduced in future revisions.

---

## 9. Error Model & Events

This section describes architecturally visible error reporting and non-architectural counters related to XMEM.

### 9.1 Architectural Events (W1C)

`CAP.XMEM.EVENTS` (and `XMIR.EVENTS` if implemented) provide sticky W1C status bits for:
- `INV_MISS`   - invalid address or cache line miss beyond specified bounds (where applicable)
- `LOCK_OVF`   - cache lock overflow or illegal lock operation
- `LDS_OOM`    - LDS allocation failure/out-of-memory
- `DMA_ERR`    - DMA/streaming engine error related to XMEM policy
- `COMP_ERR`   - compression/decompression error or unsupported mode
- `BUDGET_OVF` - LDS budget exceeded (best-effort enforcement)
- `SEC_VIOL`   - security tag violation (Section 8)
- `HYB_FALLBK` - hybrid class fallback invoked

Notes:
- Events are sticky until written with 1 (W1C). Reading does not clear bits.
- If a feature is disabled via `CAP.XMEM.CAP`, related event bits are RAZ/WI.
- Event signaling must be consistent across engines; mirrors (if present) must reflect CAP.EVENTS at instruction boundaries.

### 9.2 Faults vs. Events

- Faults (e.g., illegal instruction, access fault) follow the base ISA/engine rules; XMEM events are status only and do not mandate traps.
- When an operation is specified to trap (e.g., compressed ops when `COMPRESS=0` and implementation chooses to trap), event bits may still be set for diagnostics but do not replace trap behavior.

### 9.3 Non-Architectural Counters (Informative)

Per-engine counters such as `stall_mem`, `dma_bytes`, `lds_hits`, `l1_hits`, `l2_hits`, `streams_armed`, `lines_locked`, `cycles_busy_rt`, `cycles_busy_mtx` are informative only. If exposed:
- Document them as PMU/debug registers outside CAP.
- Do not rely on them for architectural correctness.
- Ensure counters do not interfere with CAP.XMEM state or ordering.

---

## 10. Reset & Initialization Behavior

This section describes architectural reset behavior for XMEM-related CSRs. Unless otherwise stated, all CAP.XMEM fields reset to 0 and mirrors (if present) must reflect the reset CAP state.

### 10.1 Reset Defaults (CAP-owned)

| CSR                     | Default                                 |
| ----------------------- | --------------------------------------- |
| `xmem_l1cfg.ACTIVE_CLS` | 0 (single class)                        |
| `xmem_clsmap`           | CL0=all ways, others=0                  |
| `xmem_descpol`          | DEF_CLASS=0, HYB_ENABLE=1, STRICT_DOM=0 |
| `xmem_lds_budget`       | 0 (unlimited)                           |
| All new CSRs            | Read as 0 if unimplemented              |

Notes:
- On reset, XMEM behaves as if all CAP memory registers were zero unless documented otherwise; software must not assume non-zero defaults beyond the table above.
- Feature gating via `CAP.XMEM.CAP` applies after reset; fields tied to disabled features are RAZ/WI.
- If mirrors exist, they must be synchronized to CAP on or before the first instruction boundary after reset.

### 10.2 Initialization Considerations (Informative)

- Software should program `CAP.XMEM.CLSMAP`, `L1CFG`, `L2CFG`, `DOM`, and streaming/compression defaults early in boot to avoid relying on zeroed defaults.
- Descriptor policy (`DESCPOL`) and budgets should be explicitly set before enabling workloads that depend on partitioning or hybrid flows.
- TODO: Clarify any SoC-level boot flows that pre-initialize CAP.XMEM fields prior to handing control to software in a future revision.

---

## 11. Compatibility & Interoperability

This section defines how XMEM remains consistent across domains and interoperates with other XPHMG components.

### 11.1 Cross-Domain Interoperability & Implementation Requirements (Normative)

All XPHMG domains (RSV, MTX, RT, GFX, NPU, scalar ALU) must appear to execute under a unified XMEM model defined by CAP.XMEM. These rules are mandatory unless explicitly marked as advisory.

#### 11.1.1 Architectural Source of Truth

- `CAP.XMEM.*` is the authoritative state for memory classes, domains, LDS budgets, streaming defaults/profiles, compression defaults, descriptor policies, and SVMEM configuration.
- Engine-local mirrors are permitted but must be synchronized at instruction boundaries and must not diverge architecturally.

#### 11.1.2 Instruction-Boundary Latching

- At each instruction boundary, engines latch the effective `CAP.XMEM.*` state: `L1CFG/L2CFG`, `CLSMAP/LDS_BUDGET`, `DOM`, `STREAM_CTL/SPROF*`, `COMP_CTL`, `DESCPOL`, and `SVM*` fields.
- Mid-instruction CSR writes do not affect in-progress operations.

#### 11.1.3 SVMEM Consistency

All domains issuing indexed operations (RSV, GFX, MTX tile loads, RT accesses, NPU descriptor helpers) must observe the same SVMEM semantics:
- Fields: `SVMBASE`, `SVMIDX`, `SVMSCL`, `SVMLEN`, `SVMMODE`
- Predication: masked lanes do not issue memory accesses.
- ZMODE: masked-off lanes follow `CAP.PREC.MODE.ZMODE` (merge vs zeroing).
- FOF: fault-only-first stops on the same element index for all domains.
- Element width: if `SVMLEN=0`, derive from `CAP.PREC.STAT.EFF_EW`.

#### 11.1.4 Streaming Profiles & Prefetch

- Defaults come from `CAP.XMEM.STREAM_CTL`; descriptor `sprof_id` may override when non-zero.
- Prefetch depth/stride and non-temporal policy must not diverge across domains for the same profile.

#### 11.1.5 Memory Classes & Domains

- `CAP.XMEM.DOM` defines default coherence behavior; `CAP.XMEM.CLSMAP` defines mapping of CL0-CL3 to ways/QoS.
- `CAP.XMEM.LDS_BUDGET` provides soft class budgets.
- Domains must not redefine class/domain mapping rules.

#### 11.1.6 LDS Behavior

- `XLDS.ALLOC/FREE` and `XLDS.BAR.*` apply uniformly across domains.
- LDS alignment/layout rules must not vary per domain.

#### 11.1.7 Descriptor Integration (`XPHMG_MEMCTL`)

- Descriptor fields override CAP defaults only when explicitly set: `dom`, `mclass`, `sprof_id`, `hyb`, `sec`.
- Unspecified fields fall back to `CAP.XMEM.DESCPOL` or other CAP defaults.

#### 11.1.8 Compression Consistency

- When `CAP.XMEM.COMP_CTL` enables compression, all domains use the same default mode/dictionary.
- If a domain lacks compression support, it must ignore compressed hints and issue uncompressed accesses while setting `COMP_ERR` in `CAP.XMEM.EVENTS`.

#### 11.1.9 Error & Event Reporting

- All domains report to `CAP.XMEM.EVENTS` for OOB/misaligned accesses, LDS exhaustion, lock overflows, compression failures, security violations, and hybrid fallback.

#### 11.1.10 Deterministic Replay / Debug

- Debuggers must observe latched XMEM state per instruction boundary.
- Trace tools should present XMEM/SVMEM state synchronized with CAP.PREC state.
- Engines must not expose partially updated XMEM configurations.

#### 11.1.11 Summary (Normative)

- Determinism: no mid-instruction effects.
- Cross-domain consistency: unified SVMEM, ZMODE, FOF, streaming, LDS, class mapping.
- Descriptor coherence: per-job overrides limited to documented fields.
- Correctness: deviations must raise a memory-event sticky.
- Portability: pipelines (RT->MTX->RSV->GFX->NPU) remain valid on all XPHMG implementations.

### 11.2 Compatibility

- Precision and numeric control are fully delegated to `XPHMG_PREC`.
- Features missing from `xmem_cap` may be ignored without functional impact.
- Legacy `XMEM.DMA` / `XMEM.FENCE` opcodes are deprecated and reserved.
- All previously defined opcodes (`XLDS.*`, `XMEM.LDG/STG/PREF/CCTL`, `XSWZ.*`, `XMEM.DLOADC/DSTOREC`) remain; only the configuration source is CAP.

---

## 12. Programming Considerations / Profiles (Informative)

This section provides advisory profiles and configuration patterns. No new architectural requirements are introduced.

### 12.1 Example Profiles

| Use Case             | Recommended Setting                                              |
| -------------------- | ---------------------------------------------------------------- |
| **NPU-only**         | `CLASS_PART=0`, single CL0.                                      |
| **RTU-only**         | `CLASS_PART=0`, single CL0.                                      |
| **Hybrid (NPU+RTU)** | `CLASS_PART=1`, `ACTIVE_CLS=2` or `3`. CL0=RT, CL1=MTX, CL2=HYB. |
| **Unified**          | Same as Hybrid; scheduler merges queues under CL2.               |

NOTE: advisory programming considerations; keep normative rules in Section 11.

### 12.2 Configuration Guidelines (Informative)

- Explicitly program `CAP.XMEM.CLSMAP`, `L1CFG`, `L2CFG`, and `DOM` during initialization; do not rely on reset defaults unless documented.
- Use descriptor overrides (`mclass`, `sprof_id`, `dom`, `hyb`, `sec`) sparingly and only when non-zero fields are needed; defaults come from CAP.XMEM.
- Select scopes for fences (`XLDS.BAR`, `XFENCE.*`) consistent with the intended domain visibility; global fences (`ff=global`) should be used when DMA/streams must be ordered.

### 12.3 Notes for Implementers (Informative)

- Profiles are illustrative only; conformance is defined by Sections 3-11.
- Document any platform-specific pre-configuration of CAP.XMEM (e.g., boot ROM programming) so software does not make incorrect assumptions.
- TODO: Clarify additional domain/class recipes if future revisions add more classes or domain IDs.

---

## 13. Data Layout Model (XDL) (Informative)

This section defines a common vocabulary and abstract model for how data is laid out inside XMEM regions. Tiers (T0/T1/...) define where data resides and its latency/visibility properties; XDL defines how data is organized in memory. XDL is informative and does not introduce new opcodes.

### 13.1 Goals and Scope

XDL is designed to:

- Provide a shared conceptual model for MTX, GFX, RT, RSV and ACCEL data layouts.
- Enable portable runtimes and compilers to reason about tiling, alignment and swizzles.
- Avoid overspecifying hardware implementation details (concrete encodings remain vendor-defined unless stated otherwise).

XDL does **not** define new instructions; it is consumed by engines via:

- `XPHMG_MEMCTL` metadata (e.g., `mclass`, optional layout identifiers),
- XMEM/XSWZ controls,
- Engine-specific ISA descriptions that reference XDL concepts (e.g., "operates on tiles defined by the XDL tile layout").

### 13.2 Core XDL Concepts

XDL defines the following abstract entities:

- **Element** - the smallest scalar unit (e.g., FP16, BF16, FP8, INT8, INT32).
- **Lane** - a sequence of elements processed as a contiguous vector (e.g., RSV lanes).
- **Tile** - a 2D block of elements, typically `MxN`, used by MTX and tiled GFX.
- **Tensor** - an N-dimensional array (N >= 1), with dimension names such as N, C, H, W, or generic D0..Dn.
- **Surface** - a 2D/2.5D GPU-like resource (e.g., render targets, G-buffers, textures).
- **Node/Record** - structured records used by RT (BVH nodes, triangles, hit records, etc.).

Engines may interpret the same XMEM region as different XDL entities depending on `mclass` and engine mode.

### 13.3 Alignment and Granularity

At minimum, implementations MUST guarantee:

- A **minimal alignment** for XMEM regions sufficient to hold at least one tile or record (implementation-defined, but typically >= 16 bytes).
- That elements forming a Lane, Tile, Tensor slice or Node are not split across strictly non-contiguous address spaces.

Further alignment constraints MAY be defined per `mclass` (e.g., BVH nodes aligned to 32 or 64 bytes, MTX tiles aligned to cachelines, etc.).

NOTE: Alignment requirements beyond the minimum are implementation-defined and must be documented; software should assume only the stated minimum unless a specific `mclass` defines stricter alignment.

### 13.4 Canonical XDL Tile Layout (`XDL_TILE4x4`)

XPHMG defines a mandatory, canonical tile layout that is shared across MTX, RSV, GFX and RT engines when operating on tile-shaped data via XMEM:

**`XDL_TILE4x4_ROWMAJOR`**  
A 4x4 tile of elements laid out in row-major order:

```

t[0][0], t[0][1], t[0][2], t[0][3],
t[1][0], t[1][1], t[1][2], t[1][3],
t[2][0], t[2][1], t[2][2], t[2][3],
t[3][0], t[3][1], t[3][2], t[3][3]

```

Properties:

- Mandatory for all XPHMG implementations.  
- Tiles MUST be element-contiguous and stride-aligned (>= 16 bytes, implementation MAY require higher alignment).  
- Engines MAY reinterpret the tile in multiple views:
  - **Row-vector view:** 4 vectors of length 4  
  - **Column-vector view:** 4 vectors with stride  
  - **Flat view:** 1 vector of 16 elements  
- All cross-engine handoffs in Tier-0 MUST use this layout unless otherwise specified by `mclass` or explicit layout metadata.

### 13.5 Composite Tiles (`Linked Tiles`)

Larger logical tiles may be constructed from multiple canonical 4x4 tiles placed contiguously in memory.  
This mechanism allows flexible tile sizes (e.g., 4x8, 8x4, 8x8, 4x16...) without requiring MTX hardware to natively support those shapes.

A composite tile is defined as:

```
XDL_TILE(M,N) = linkage of (M/4) x (N/4) canonical 4x4 tiles
in row-major tile order.
```

Example:

- `XDL_TILE4x8` consists of two adjacent `XDL_TILE4x4_ROWMAJOR` tiles.  
- `XDL_TILE8x8` consists of four 4x4 tiles arranged as a 2x2 block.

Rules:

- Engines MAY operate on large composite tiles by issuing multiple operations on the underlying 4x4 base tiles.  
- Engines that implement larger tiles natively MAY fuse these operations internally.  
- The architectural result MUST match the composite-tile definition regardless of internal implementation.

This model allows:
- Simple implementations (processing one 4x4 tile at a time)  
- Advanced implementations (processing 4 tiles in parallel)  
without changing the ISA.

NOTE: Composite tiles are defined purely by logical concatenation of canonical tiles; internal hardware fusion or scheduling does not change the architectural layout or element order.


### 13.6 Layout Identification (Conceptual)

Implementations MAY associate layout metadata with an XMEM region, either:

- implicitly via `mclass`, or
- explicitly via a layout identifier (`layout_id`) associated with the descriptor (encoding is vendor-defined unless standardized in future revisions).

Examples of layout categories:

- **Scalar / Linear** - simple 1D arrays.
- **SoA / AoS / AoSoA** - struct-of-arrays / array-of-structs hybrids.
- **Tiled** - fixed-size tiles (e.g., 4x4, 8x8) with row-major or column-major ordering.
- **Swizzled** - Morton/Z-order, block-compressed, or GPU-style texture layouts.
- **Tensor** - common ML layouts (NCHW, NHWC, planar, interleaved).

The XDL model only standardizes *names and semantics* of these categories; concrete encodings, such as numeric `layout_id` values, are left to implementers unless otherwise standardized.

### 13.7 Relationship Between XDL and Tiers

- XDL is **tier-agnostic**: a given layout (e.g., tiled 4x4 FP16) can be used in Tier-0, Tier-1, or higher tiers.
- In practice:
  - Tier-0 is typically used for *small, already well-formed* tiles / records ready for consumption by engines.
  - Tier-1 is often used to **construct and transform XDL entities** (e.g., re-tile, re-layout, gather/scatter staging).
  - Tier-2+ is used for long-term storage, large tensors, and streaming sources/sinks.

Engines SHOULD document which XDL categories they natively support for optimal performance (e.g., MTX supports fixed-size tiles; RT supports specific BVH node layouts; GFX supports specific surface swizzles).

NOTE: When moving data across tiers or domains (e.g., Tier-1 to Tier-0 handoff), layout metadata must remain consistent; engines must not reinterpret a declared layout.

### 13.8 Future Standardization

Future revisions of this specification MAY:

- Define a small set of standardized `layout_id` values for common cases (e.g., a canonical MTX tile format, a canonical BVH node format, common tensor layouts).
- Introduce optional XMEM/XSWZ helpers that transform between XDL layouts (e.g., linear -> tiled, SoA -> AoS, Morton -> linear).

Until then, XDL serves as the common language across XPHMG extensions, allowing engines and runtimes to coordinate without hardcoding vendor-specific layout assumptions into each ISA document.

---

## 14. Examples (Informative Appendix)

The following examples illustrate how CAP.XMEM state and descriptor overrides drive behavior across domains. They are informative and do not add new requirements.

### 14.1  RTU -> MTX Denoiser Chain (using CAP state)

```asm
# Configure CAP.XMEM.CLSMAP: CL0=RT (3 ways), CL1=MTX (5 ways)
li   t0, (0b00000111) | (0b11100000 << 8)
csrw CAP.XMEM.CLSMAP, t0

# Optionally update XMIR mirror if aliased
# csrw XMIR.CLSMAP, t0

# --- RT job ---
li   t1, DOM_RT
csrw CAP.XMEM.DOM, t1
memctl.dom    = DOM_RT
memctl.mclass = 0
xsubmit.rt t0, &desc_rt

xfence.signal 1

# --- MTX job ---
xfence.wait 1
li   t1, DOM_MTX
csrw CAP.XMEM.DOM, t1
memctl.dom    = DOM_MTX
memctl.mclass = 1
xsubmit.mtx t1, &desc_mtx
```

### 14.2  Unified Hybrid Task (Shared LDS)

```asm
# Hybrid / Unified domain
li   t0, DOM_ALL
csrw CAP.XMEM.DOM, t0

# Enable Hybrid sharing through CAP defaults
li   t1, ((1<<2) | (1<<0))  # HYB_ENABLE=1, DEF_CLASS=0
csrw CAP.XMEM.DESCPOL, t1

memctl.dom    = DOM_ALL
memctl.mclass = 2     # CL2 = HYB
memctl.hyb    = 1
xsubmit.mtx t0, &desc_unified
```

> **Implementation note:** If `XMIR.*` exists and operates in aliased mirror mode, software may program the same values through `XMIR.*`; both interfaces are architecturally equivalent at decode time. Mirrors must match CAP at instruction boundaries.

---

## 15. Normative Statement Summary

This appendix lists key normative statements consolidated from the specification for quick reference.

- **N-1. Instruction-boundary latching:** Engines must observe the latest `CAP.XMEM.*` state at instruction boundary; mirrors (if any) must be coherent with CAP at that boundary.
- **N-2. Capability gating:** If a feature bit in `CAP.XMEM.CAP` is clear, requests targeting that feature are ignored safely; associated CSRs read as zero and writes are ignored (RAZ/WI).
- **N-3. SVMEM predication/FOF:** Gather/scatter must honor RSV predication and `CAP.PREC.MODE.ZMODE`; with `FOF=1`, the first faulting element index must be written to `RSV.SVFAULTI` and the operation aborted.
- **N-4. LDS budgets and classes:** LDS allocation and streaming policies must enforce CAP budgets and classes on a best-effort basis, reporting overflows via `CAP.XMEM.EVENTS`.
- **N-5. Descriptor precedence:** Descriptor fields override CAP defaults only when non-zero and supported; otherwise CAP defaults apply.
- **N-6. Cross-domain consistency:** All domains must use the same meanings for `dom`, `mclass`, streaming profiles, compression defaults, and SVMEM semantics defined by CAP.
- **N-7. Ordering scopes:** Fence/barrier scopes follow the hierarchy in Section 3.1; ordering is with respect to the current `dom` and class selection.
- **N-8. Security (optional):** If `SEC_DOM=1`, accesses violating security tags must raise `SEC_VIOL`; if `SEC_DOM=0`, security tags are RAZ/WI and must not fault.
- **N-9. Compression fallback:** If a domain does not support inline compression, it must ignore compressed hints and issue uncompressed accesses while setting `COMP_ERR` in `CAP.XMEM.EVENTS`.

---
Designed for integration with `XPHMG_RSV`, `XPHMG_MTX`, `XPHMG_RT`, and `XPHMG_CAP`.

*Licensed under CC-BY-SA 4.0 - Open Specification maintained by Pedro H. M. Garcia.*