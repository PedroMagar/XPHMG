# RISC-V Vendor Extension **XPHMG_XMEM**
**Unified LDS / Cache / Streaming Memory Control**

**Category:** Vendor Extension (part of `XPHMG_*` family)  
**Version:** 0.1.0
**Author:** Pedro H. M. Garcia  
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview (Informative)

`XPHMG_XMEM` defines the **explicit memory control model** shared by all XPHMG execution domains.

Rather than exposing implicit cache behavior or driver-managed heuristics, XMEM provides architectural control over:

* Local Data Store (LDS) allocation and lifetime,
* Cache and streaming behavior,
* Memory classes and quality-of-service,
* Coherence domains and cross-domain visibility,
* Indexed gather/scatter (SVMEM),
* Descriptor-driven memory policy overrides.

XMEM does not own policy itself.
All architectural defaults, limits, and capabilities are defined in `CAP.XMEM.*`; XMEM instructions and descriptors consume this state explicitly.

XMEM enables predictable, composable memory behavior across RSV, MTX, RT, GFX, and ACCEL domains without introducing implicit promotion, eviction, or background processing.

---

## 2. Memory hierarchy model

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

---

## 3. Engine-Local Mirror CSRs (non-architectural)

These CSRs are **optional vendor mirrors** used for fast reprogramming or debug.
They **MUST** reflect the canonical architectural state stored in `CAP.XMEM.*` at every instruction boundary.
Software must consider **`CAP.XMEM.*` as the single source of truth**; the `XMIR.*` registers are implementation conveniences only.

**N-0 (Mirror Coherence).**
Immediately before any memory instruction issues, the mirror **must equal** the corresponding `CAP.XMEM.*` value.
If the platform supports *deferred apply*, mirrors must synchronize before the next instruction with side effects.

---

### Architectural Defaults

Unless overridden by a descriptor or instruction immediate, engines **must use** the following `CAP.XMEM.*` state:

* `CAP.XMEM.L1CFG`, `CAP.XMEM.L2CFG` — cache line size, replacement, locking, active class
* `CAP.XMEM.CLSMAP`, `CAP.XMEM.LDS_BUDGET` — class/way mapping and LDS soft budgets
* `CAP.XMEM.DOM` — default coherence domain
* `CAP.XMEM.STREAM_CTL`, `CAP.XMEM.SPROF[0..3]` — streaming detector/profiles
* `CAP.XMEM.COMP_CTL` — inline-compression defaults
* `CAP.XMEM.DESCPOL` — descriptor fallback policies
* `CAP.XMEM.SVMBASE/SVMIDX/SVMSCL/SVMLEN/SVMMODE` — gather/scatter control
* `CAP.XMEM.EVENTS` — sticky status (W1C)

**Notes**

* If a feature bit in `CAP.XMEM.CAP` = 0, the corresponding behavior is architecturally disabled; the mirror must read as 0 and ignore writes.
* Descriptor fields (`mclass`, `sprof_id`, `dom`, `sec`) override CAP defaults only when non-zero and supported.
* Implementations may choose **Aliased Mirror Mode** (writes propagate to CAP) or **Read-Only Mirror Mode** (writes ignored — RAZ/WI).
  The chosen mode **must** be documented.

---

### 3.1  XMIR.LDSZ (RO)

Mirror of `CAP.XMEM.PARAMS.LDS_KiB`.
Reports total LDS bytes per cluster.

### 3.2  XMIR.LDSPART (RW)

Snapshot of active LDS allocations.
Read-only if hardware auto-partitions.

### 3.3  XMIR.L1CFG (RW)  — Mirror of `CAP.XMEM.L1CFG`

|  Bits | Field        | Description                                       |
| :---: | :----------- | :------------------------------------------------ |
|  7:0  | `WAYS`       | Associativity (hint if fixed)                     |
|  15:8 | `LINE_SZ`    | Sector size (0=32 B, 1=64 B, 2=128 B, 3=impl.)    |
| 19:16 | `REPL`       | Replacement (0 LRU · 1 PLRU · 2 LFU · 3 RR/impl.) |
|   20  | `LOCK_EN`    | Enable line/way lock ops                          |
| 31:24 | `ACTIVE_CLS` | Active class index (0–3)                          |

### 3.4  XMIR.CLSMAP (RW)  — Mirror of `CAP.XMEM.CLSMAP`

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

*Default mapping recommendation:*  CL0 = RT · CL1 = MTX · CL2 = HYB · CL3 = OTHER.

### 3.5  XMIR.LDS_BUDGET (RW)  — Mirror of `CAP.XMEM.LDS_BUDGET`

| Bits | Field | Description |
| 15:0 | `CL0_KiB` | Budget for CL0 |
| 31:16 | `CL1_KiB` | Budget for CL1 |
| 47:32 | `CL2_KiB` | Budget for CL2 |
| 63:48 | `CL3_KiB` | Budget for CL3 |

Overflow → `CAP.XMEM.EVENTS.BUDGET_OVF`.

### 3.6  XMIR.L2CFG (RW)  — Mirror of `CAP.XMEM.L2CFG`

Victim policy, QoS, snoop coherence, and compression threshold.

### 3.7  XMIR.DOM (RW)  — Mirror of `CAP.XMEM.DOM`

Default domain: `1=CPU  2=RT  3=MTX  15=ALL`.

### 3.8  XMIR.STREAM_CTL (RW)  — Mirror of `CAP.XMEM.STREAM_CTL`

Fields: `stride`, `pf_depth`, `nontemp_en`.

### 3.9  XMIR.SPROF[0–3] (RW)  — Mirror of `CAP.XMEM.SPROF[0–3]`

Each profile: `stride[15:0]`, `pf_depth[23:16]`, `nt_policy[27:24]`.

### 3.10  XMIR.COMP_CTL (RW)  — Mirror of `CAP.XMEM.COMP_CTL`

Compression mode/level/dictionary for inline compression.

### 3.11  XMIR.DESCPOL (RW)  — Mirror of `CAP.XMEM.DESCPOL`

| Bits | Field | Description |
| 1:0 | `DEF_CLASS` | Default class for jobs without `mclass` |
| 2 | `HYB_ENABLE` | Allow HYB (CL2) borrowing |
| 5:3 | `HYB_BORROW_QOS` | Borrow priority |
| 6 | `STRICT_DOM` | Reject domain mismatch |
| 7 | `RSV` | Reserved |

### 3.12  XMIR.EVENTS (W1C Mirror of `CAP.XMEM.EVENTS`)

Sticky flags: `INV_MISS`, `LOCK_OVF`, `LDS_OOM`, `DMA_ERR`, `COMP_ERR`, `BUDGET_OVF`, `SEC_VIOL`, `HYB_FALLBK`.

### 3.13 Shared Hot Data and Tier-0 Semantics (XMEM-Level)

This section formalizes how XMEM represents, allocates, partitions and transfers Tier-0 data across engines.

#### 3.13.1 Unified XMEM View

All engines (RSV, MTX, GFX, RT, ACCEL) view XMEM through the same descriptor format `XPHMG_MEMCTL`.  
Direct engine-to-engine register forwarding is not architecturally defined; all cross-engine dataflow MUST occur through XMEM regions.

#### 3.13.2 Memory Class for Hot Shared Data (`CLS_SHARED`)

Implementations MUST reserve at least one memory class ID for shared hot data:

- **`CLS_SHARED`** — identifies XMEM regions intended for cross-engine communication using Tier-0.

Other memory classes remain implementation-defined (e.g., `CLS_GBUF`, `CLS_MTX`, `CLS_RT`, etc.).

#### 3.13.3 Tier Field (`tier`) in `XPHMG_MEMCTL`

`XPHMG_MEMCTL` SHALL include a `tier` field (min. 2 bits) that indicates the intended tier of an allocated region:

| tier value | Meaning |
|-----------|---------|
| `T0` | Tier-0 / shared hot data (quasi-register) |
| `T1` | Regular LDS/XMEM |
| `T2` | Coherent memory / higher cache tiers |
| `T3` | DRAM / external memory |

A region marked as `CLS_SHARED / T0` is eligible for cross-engine handoff.

#### 3.13.4 Logical Ownership (`dom`)

`dom` indicates which engine currently owns or has priority access to the region:

- `DOM_MTX`, `DOM_RT`, `DOM_GFX`, `DOM_RSV`, `DOM_ACCEL`,  
- `DOM_CPU`,  
- or `DOM_ALL` for shared read access.

Changing `dom` on a `CLS_SHARED / T0` region MUST NOT imply physical data movement.  
It is strictly a logical ownership transition used to orchestrate pipeline stages (e.g., MTX→RT→GFX).

#### 3.13.5 Partitioning and Tier-0 Mapping (`XMEM.CCTL.PARTITION`)

`XMEM.CCTL.PARTITION` MUST describe which physical subrange(s) of LDS/XMEM are reserved for Tier-0 use.  
Only regions whose `tier` field equals `T0` MAY map into this Tier-0 partition.

Tier-0 regions:

- MUST provide lower-latency access than Tier-1,  
- MAY have size limitations,  
- MAY restrict aliasing,  
- MAY implement additional hazard or fencing semantics.

#### 3.13.6 Handoff Semantics

A handoff between engines (e.g., MTX→RT) is architecturally defined as:

```

1. Producer writes to region R marked as:
   R = CLS_SHARED / T0 / DOM_PRODUCER
2. Upon completion, software or micro-scheduler updates:
   R.dom = DOM_CONSUMER
3. Consumer reads region R without further data movement.

```

No explicit synchronization barriers are implied beyond normal XMEM ordering rules, unless otherwise specified by the engine’s ISA extension.

#### 3.13.7 Engine Behavior

- Engines MUST treat any `CLS_SHARED / T0` region with matching `dom` as a valid hot data source or sink.  
- Engines MUST NOT read or write Tier-0 regions whose `dom` does not authorize access.  
- Engines MAY apply optimizations (prefetch, low-latency paths, co-issue) specifically for Tier-0.

#### 3.13.8 Fallback to Tier-1

If Tier-0 capacity is insufficient, engines MAY allocate regions as:

- `CLS_SHARED / T1 / DOM_*`

Such regions still participate in the unified dataflow model, but may incur additional latency and may require explicit synchronization barriers.

#### 3.13.9 Tier-1 Regions and Working Sets

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

##### 3.13.9.1 Access Semantics

- Tier-1 regions follow standard XMEM ordering rules.
- Engines MAY freely allocate, read and write Tier-1 regions, subject to `dom` permissions.
- Unlike Tier-0, Tier-1 **does not** imply special hot-handoff semantics. Changing `dom` on a Tier-1 region does **not** carry the same implicit "handoff" meaning; it is a policy hint for arbitration and scheduling.

##### 3.13.9.2 Relationship to Tier-0 and Tier-2+

- Tier-1 sits between Tier-0 and Tier-2:

  - Tier-0: shared hot data (`CLS_SHARED / T0`) used as quasi-register for cross-engine handoff.
  - Tier-1: larger working sets, with LDS/XMEM latency and capacity.
  - Tier-2+: coherent caches / DRAM, not directly controlled by XPHMG.

- A typical pattern is:

  1. Data is constructed, re-tiled, or transformed in Tier-1.
  2. A subset is promoted or projected into Tier-0 (`CLS_SHARED / T0`) for cross-engine consumption.
  3. Results may be written back to Tier-1 or to higher tiers for long-term storage.

##### 3.13.9.3 Partitioning and QoS

`XMEM.CCTL.PARTITION` MAY also expose how Tier-1 capacity is partitioned across memory classes and engines, e.g.:

- Preferred size per `mclass` (e.g., `CLS_GBUF`, `CLS_RT`, `CLS_MTX`).
- Engine-local vs. shared Tier-1 pools.
- Optional QoS hints (priority levels) for Tier-1 allocations.

These details are implementation-defined but MUST NOT change the architectural semantics: a region marked as `tier = T1` MUST preserve LDS/XMEM-like latency and be accessible by any engine that holds appropriate `dom` permissions.

### 3.13.10 XDL Tiles in Tier-0 and Tier-1

When a region is marked as `CLS_SHARED / T0`, and its layout metadata corresponds to `XDL_TILE4x4_ROWMAJOR` or a composite tile, engines MUST interpret the memory content as an XDL tile suitable for cross-engine consumption.

- Tier-0 regions are intended for **hot tiles** produced by MTX and consumed by RSV/GFX/RT.  
- Tier-1 regions are the recommended location for constructing and linking composite tiles.

No engine is allowed to reinterpret the tile using a non-XDL layout when the region declares an XDL tile layout.

### 3.13.11 Tier-2 and Tier-3 Regions

While Tier-0 and Tier-1 are explicitly modeled as LDS/XMEM-backed regions, Tier-2 and Tier-3 represent how such regions are serviced by the wider memory system:

- **Tier-2 (T2)** — Regions whose primary residency is expected to be within the coherent cache hierarchy (L1/L2/fabric-level shared memory).
- **Tier-3 (T3)** — Regions whose primary residency is in external DRAM or equivalent high-latency memory.

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
- Implementations MAY map T2/T3 to one or more of the memory levels described in §2 (L1$/L2$/XDMA/UMF), but no particular mapping is mandated.

---

## 4. XPHMG Data Layout Model (XDL)

This section defines a common vocabulary and abstract model for how data is laid out inside XMEM regions.  
Tiers (T0/T1/…) define *where* data resides and its latency/visibility properties; XDL defines *how* data is organized in memory.

### 4.1 Goals and Scope

XDL is designed to:

- Provide a shared conceptual model for MTX, GFX, RT, RSV and ACCEL data layouts.
- Enable portable runtimes and compilers to reason about tiling, alignment and swizzles.
- Avoid overspecifying hardware implementation details (concrete encodings remain vendor-defined unless stated otherwise).

XDL does **not** define new instructions; it is consumed by engines via:

- `XPHMG_MEMCTL` metadata (e.g., `mclass`, optional layout identifiers),
- XMEM/XSWZ controls,
- Engine-specific ISA descriptions that reference XDL concepts (e.g., "operates on tiles defined by the XDL tile layout").

### 4.2 Core XDL Concepts

XDL defines the following abstract entities:

- **Element** — the smallest scalar unit (e.g., FP16, BF16, FP8, INT8, INT32).
- **Lane** — a sequence of elements processed as a contiguous vector (e.g., RSV lanes).
- **Tile** — a 2D block of elements, typically `M×N`, used by MTX and tiled GFX.
- **Tensor** — an N-dimensional array (N ≥ 1), with dimension names such as N, C, H, W, or generic D0..Dn.
- **Surface** — a 2D/2.5D GPU-like resource (e.g., render targets, G-buffers, textures).
- **Node/Record** — structured records used by RT (BVH nodes, triangles, hit records, etc.).

Engines may interpret the same XMEM region as different XDL entities depending on `mclass` and engine mode.

### 4.3 Alignment and Granularity

At minimum, implementations MUST guarantee:

- A **minimal alignment** for XMEM regions sufficient to hold at least one tile or record (implementation-defined, but typically ≥ 16 bytes).
- That elements forming a Lane, Tile, Tensor slice or Node are not split across strictly non-contiguous address spaces.

Further alignment constraints MAY be defined per `mclass` (e.g., BVH nodes aligned to 32 or 64 bytes, MTX tiles aligned to cachelines, etc.).

### 4.4 Canonical XDL Tile Layout (`XDL_TILE4x4`)

XPHMG defines a mandatory, canonical tile layout that is shared across MTX, RSV, GFX and RT engines when operating on tile-shaped data via XMEM:

**`XDL_TILE4x4_ROWMAJOR`**  
A 4×4 tile of elements laid out in row-major order:

```

t[0][0], t[0][1], t[0][2], t[0][3],
t[1][0], t[1][1], t[1][2], t[1][3],
t[2][0], t[2][1], t[2][2], t[2][3],
t[3][0], t[3][1], t[3][2], t[3][3]

```

Properties:

- Mandatory for all XPHMG implementations.  
- Tiles MUST be element-contiguous and stride-aligned (≥ 16 bytes, implementation MAY require higher alignment).  
- Engines MAY reinterpret the tile in multiple views:
  - **Row-vector view:** 4 vectors of length 4  
  - **Column-vector view:** 4 vectors with stride  
  - **Flat view:** 1 vector of 16 elements  
- All cross-engine handoffs in Tier-0 MUST use this layout unless otherwise specified by `mclass` or explicit layout metadata.

### 4.5 Composite Tiles (`Linked Tiles`)

Larger logical tiles may be constructed from multiple canonical 4×4 tiles placed contiguously in memory.  
This mechanism allows flexible tile sizes (e.g., 4×8, 8×4, 8×8, 4×16…) without requiring MTX hardware to natively support those shapes.

A composite tile is defined as:

```
XDL_TILE(M,N) ≡ linkage of (M/4) × (N/4) canonical 4×4 tiles
in row-major tile order.
```

Example:

- `XDL_TILE4x8` consists of two adjacent `XDL_TILE4x4_ROWMAJOR` tiles.  
- `XDL_TILE8x8` consists of four 4×4 tiles arranged as a 2×2 block.

Rules:

- Engines MAY operate on large composite tiles by issuing multiple operations on the underlying 4×4 base tiles.  
- Engines that implement larger tiles natively MAY fuse these operations internally.  
- The architectural result MUST match the composite-tile definition regardless of internal implementation.

This model allows:
- Simple implementations (processing one 4×4 tile at a time)  
- Advanced implementations (processing 4 tiles in parallel)  
without changing the ISA.


### 4.6 Layout Identification (Conceptual)

Implementations MAY associate layout metadata with an XMEM region, either:

- implicitly via `mclass`, or
- explicitly via a layout identifier (`layout_id`) associated with the descriptor (encoding is vendor-defined unless standardized in future revisions).

Examples of layout categories:

- **Scalar / Linear** — simple 1D arrays.
- **SoA / AoS / AoSoA** — struct-of-arrays / array-of-structs hybrids.
- **Tiled** — fixed-size tiles (e.g., 4×4, 8×8) with row-major or column-major ordering.
- **Swizzled** — Morton/Z-order, block-compressed, or GPU-style texture layouts.
- **Tensor** — common ML layouts (NCHW, NHWC, planar, interleaved).

The XDL model only standardizes *names and semantics* of these categories; concrete encodings, such as numeric `layout_id` values, are left to implementers unless otherwise standardized.

### 4.7 Relationship Between XDL and Tiers

- XDL is **tier-agnostic**: a given layout (e.g., tiled 4×4 FP16) can be used in Tier-0, Tier-1, or higher tiers.
- In practice:
  - Tier-0 is typically used for *small, already well-formed* tiles / records ready for consumption by engines.
  - Tier-1 is often used to **construct and transform XDL entities** (e.g., re-tile, re-layout, gather/scatter staging).
  - Tier-2+ is used for long-term storage, large tensors, and streaming sources/sinks.

Engines SHOULD document which XDL categories they natively support for optimal performance (e.g., MTX supports fixed-size tiles; RT supports specific BVH node layouts; GFX supports specific surface swizzles).

### 4.8 Future Standardization

Future revisions of this specification MAY:

- Define a small set of standardized `layout_id` values for common cases (e.g., a canonical MTX tile format, a canonical BVH node format, common tensor layouts).
- Introduce optional XMEM/XSWZ helpers that transform between XDL layouts (e.g., linear ↔ tiled, SoA ↔ AoS, Morton ↔ linear).

Until then, XDL serves as the common language across XPHMG extensions, allowing engines and runtimes to coordinate without hardcoding vendor-specific layout assumptions into each ISA document.

---

## 5 SVMEM gather/scatter execution (ties to CAP)

SVMEM is tied to CAP.

### 5.1 Index addressing model

* **Source of control:** `CAP.XMEM.SVMBASE`, `…SVMIDX`, `…SVMSCL`, `…SVMLEN`, `…SVMMODE`.
* **Effective element size:** If `SVMLEN=0`, derive from `CAP.PREC.STAT.EFF_EW`; otherwise use `SVMLEN`.
* **Scale & sign:** `SVMSCL.SCALE ∈ {1,2,4,8,16}`, `SVMSCL.SIGNED` selects sign-extended indices.
* **Gather vs scatter:** `SVMSCL.GS` bit selects load (gather) or store (scatter).
* **FOF:** If `SVMSCL.FOF=1`, on the first faulting element *i*, store *i* into `RSV.SVFAULTI` and abort.

### 5.2 Predication and masked lanes

* Per-lane requests are enabled by the current RSV mask.
* **Masked lanes writeback policy**: obey `CAP.PREC.MODE.ZMODE` (zeroing vs merge). For gathers, zeroing writes zeros to destination; for scatters, masked lanes **do not** perform a store.

### 5.3 Address computation

```
addr[i] = SVMBASE + sext_or_zext( IDX[i] ) * SCALE
```

OOB protection is implementation-defined; if `SVMMODE.BOUND_CHECK=1` and an address is out-of-range, treat as fault (or zero if `ALLOW_OOB_ZERO=1`).

### 5.4 Memory ordering and domains

* Default domain comes from `CAP.XMEM.DOM` unless overridden by descriptor or instruction hint.
* Visibility is governed by fences (see §7). The engine must honor domain-scope fences.

---

## 6. Instruction Set Summary

* **Allocation/Free.** `XLDS.ALLOC` and `XLDS.FREE` are unchanged functionally; when `CAP.XMEM.CAP.CLASS_PART=1`, the allocation **consumes from the caller’s active class** (either from descriptor `mclass` or from `CAP.XMEM.L1CFG.ACTIVE_CLASS`).
* **Budgets.** If `CAP.XMEM.LDS_BUDGET` is non-zero for a class, the engine must apply best-effort enforcement. On overflow, set `CAP.XMEM.EVENTS.BUDGET_OVF`.
* **Bank swizzle.** If `CAP.XMEM.CAP.BIDIR_SWZ=1`, `XLDS.CPY` may swizzle in both directions as a hint; correctness must not rely on swizzle.

### 6.1 LDS instructions (`XLDS.*`)

| Mnemonic                          | Summary                                                                         |
|-----------------------------------|---------------------------------------------------------------------------------|
| **XLDS.ALLOC rd, rs_size, flags** | Allocate LDS space; return base or 0 on failure.                                |
| **XLDS.FREE rs_base**             | Release allocated LDS region.                                                   |
| **XLDS.BAR imm_scope**            | Barrier ordering LDS ops up to given scope.                                     |
| **XLDS.ATOMS op, [addr], val**    | Atomic ops on LDS.                                                              |
| **XLDS.CPY rd, rs, len, flags**   | Copy between LDS and regs/global. If `BIDIR_SWZ=1`, swizzle in both directions. |

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

### 6.3 Swizzle operations (`XSWZ.*`)
| Mnemonic                                                | Summary                             |
|---------------------------------------------------------|-------------------------------------|
| **XSWZ.TEX2TEN dst, src, layout_tex, layout_ten, tile** | Texture → tensor layout conversion. |
| **XSWZ.TEN2TEX dst, src, layout_ten, layout_tex, tile** | Tensor → texture layout conversion. |

### 6.4 Optional inline compression

| Mnemonic                               | Summary                |
|----------------------------------------|------------------------|
| **XMEM.DLOADC rd, [addr], len, mode**  | Load compressed data.  |
| **XMEM.DSTOREC [addr], rs, len, mode** | Store compressed data. |

### 6.5 Compression I/O alignment

* **Defaults** (`mode/level/dictionary`) come from `CAP.XMEM.COMP_CTL`.
* The `XMEM.DLOADC/DSTOREC` opcodes remain; if `CAP.XMEM.CAP.COMPRESS=0`, they must trap as *Illegal Instruction* or act as normal LD/ST (implementation choice, document it).

---

## 7 Engine-local mirrors

Vendors may implement non-architectural *mirror CSRs* inside XMEM (e.g., `xmem_l1cfg_mirror`) for fast reprogramming.
**Rule:** at the boundary before issuing a memory op, the mirror **must equal** CAP state, or the engine must **latch CAP** into the mirror. If the platform supports *deferred apply*, mirrors must be synchronized on the next instruction boundary before any memory side-effect.

---

## 8. Descriptor extension `XPHMG_MEMCTL`

### C structure
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
  uint8_t  mclass;      // Memory class index 0–3 (maps to xmem_clsmap)
  uint8_t  sprof_id;    // Streaming profile id 0–3
  uint8_t  hyb;         // Hybrid/shared job (1=HYB)
  uint8_t  sec;         // Security tag if SEC_DOM=1
  uint32_t rsv0;        // Reserved
};
```

### Semantics

* `mclass` maps to one of the classes CL0–CL3.
* `sprof_id` selects streaming profile (0–3).
* `hyb=1` indicates hybrid job (prefer CL2).
* `sec` ignored unless `SEC_DOM=1`.
* If unsupported fields are set, hardware ignores them safely.

---

## 9. Fences & scopes

* **Cross-domain ordering:** `XFENCE.PIPE {rt|mtx|all}` orders all writes below the domain to be visible to subsequent reads.
* **Event tokens:** `XFENCE.SIGNAL` / `XFENCE.WAIT` synchronize producers and consumers.
* **Cache visibility:** Producers writing LDS must execute `XLDS.BAR ≥CLUSTER` or `XMEM.CCTL FLUSH` before signaling consumers reading via L1/L2.
* **Keep** the fence instruction names, **add**: fence scopes **must** order with respect to **CAP-selected domains/classes**.
* **Global fence**: if `ff=global`, completion includes DMA/streams whose policy came from current CAP defaults or a descriptor override.

---

## 10. Security and isolation (optional)

If `xmem_cap.SEC_DOM=1`, every LDS allocation, cache way, or DMA stream can be tagged with `sec` (2 bits recommended):
`00=public`, `01=trusted`, `10=secure`, `11=reserved`.

Violation triggers `xmem_events.SEC_VIOL`.

---

## 11. Error model & counters

* **Sticky events:** use `CAP.XMEM.EVENTS` (W1C) for: `INV_MISS, LOCK_OVF, LDS_OOM, DMA_ERR, COMP_ERR, BUDGET_OVF, SEC_VIOL, HYB_FALLBK`.
* **Per-engine counters:** keep your informative counters (`stall_mem`, `dma_bytes`, etc.) as **non-architectural**; if exposed, document as PMU or debug registers, not CAP.

**Counters (suggested):**
`stall_mem`, `dma_bytes`, `lds_hits`, `l1_hits`, `l2_hits`, `streams_armed`, `lines_locked`, `cycles_busy_rt`, `cycles_busy_mtx`.

---

## 12. Reset defaults

| CSR                     | Default                                 |
| ----------------------- | --------------------------------------- |
| `xmem_l1cfg.ACTIVE_CLS` | 0 (single class)                        |
| `xmem_clsmap`           | CL0=all ways, others=0                  |
| `xmem_descpol`          | DEF_CLASS=0, HYB_ENABLE=1, STRICT_DOM=0 |
| `xmem_lds_budget`       | 0 (unlimited)                           |
| All new CSRs            | Read as 0 if unimplemented              |

* On reset, XMEM behaves as if all CAP memory registers were zero (implementation defaults). Engines must not assume any non-zero default unless documented under CAP.

---

## 13. Domain/class interoperability

| Use Case             | Recommended Setting                                              |
| -------------------- | ---------------------------------------------------------------- |
| **NPU-only**         | `CLASS_PART=0`, single CL0.                                      |
| **RTU-only**         | `CLASS_PART=0`, single CL0.                                      |
| **Hybrid (NPU+RTU)** | `CLASS_PART=1`, `ACTIVE_CLS=2` or `3`. CL0=RT, CL1=MTX, CL2=HYB. |
| **Unified**          | Same as Hybrid; scheduler merges queues under CL2.               |

---

## 14. Worked examples

### 14.1  RTU → MTX Denoiser Chain (using CAP state)

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

> **Implementation note:** If `XMIR.*` exists and operates in *aliased mirror* mode,
> software may program the same values through `XMIR.*`; both interfaces are architecturally equivalent at decode time.

---

## **15. Cross-Domain Interoperability & Implementation Requirements (Normative)**

This section defines the rules required to ensure that **all XPHMG execution domains** (RSV, MTX, RT, GFX, NPU, scalar ALU) interact with the **XPHMG_XMEM** memory model in a consistent, deterministic, and performance-portable manner.
These rules are **mandatory** unless explicitly marked as advisory.

### **15.1 Architectural Source of Truth**

All XPHMG engines must treat the **CAP-owned XMEM CSRs** (`CAP.XMEM.*`) as the **architectural source of truth** for:

* memory classes (CL0–CL3)
* coherence domains (`DOM`)
* LDS budgets and class assignment
* streaming defaults and per-profile descriptors (`STREAM_CTL`, `SPROF0..3`)
* inline compression defaults (`COMP_CTL`)
* descriptor defaults (`DESCPOL`)
* indexed gather/scatter parameters (`SVMBASE`, `SVMIDX`, `SVMSCL`, `SVMLEN`, `SVMMODE`)

Engine-local mirrors (e.g., cached views of XMEM settings) **are permitted**, but they must always be synchronized at the **instruction boundary** and never produce architecturally visible divergence.

### **15.2 Instruction-Boundary Latching**

At each instruction boundary, every domain must latch the complete effective memory state:

* `CAP.XMEM.L1CFG`, `L2CFG`
* `CAP.XMEM.CLSMAP`, `LDS_BUDGET`
* `CAP.XMEM.DOM`
* `CAP.XMEM.STREAM_CTL`, `SPROF0..3`
* `CAP.XMEM.COMP_CTL`
* `CAP.XMEM.DESCPOL`
* `CAP.XMEM.SVM*` (gather/scatter configuration)

No mid-instruction write to any CAP.XMEM CSR may affect an in-progress operation.

This is critical for:

* RSV gather/scatter correctness
* RT BVH tile visibility and queue behavior
* MTX tile loads with streaming prefetch
* GFX texture/tensor operations
* NPU descriptor-driven DMA phases

### **15.3 SVMEM Unification (Gather/Scatter Rules)**

All domains that touch memory through indexed operations — including RSV, GFX, MTX tile loads, RT leaf/triangle accesses, and NPU descriptor helpers — must use the **same SVMEM semantics**, defined solely by:

* `SVMBASE` (base pointer)
* `SVMIDX` (index source register/pointer)
* `SVMSCL` (scale, sign, gather/scatter mode, FOF)
* `SVMLEN` (element size override)
* `SVMMODE` (bounds check, zero-on-OOB, compressed/expanded modes)

#### **15.3.1 Mandatory Consistency Rules**

1. **Predication:** masked lanes must not issue memory accesses.
2. **ZMODE:** masked-off lanes must follow `CAP.PREC.MODE.ZMODE`

   * `0` → merge (destination preserved)
   * `1` → zeroing (destination forced to 0)
3. **FOF behavior:** fault-only-first must stop on the same lane index for all domains.
4. **Element width:** if `SVMLEN=0`, the element width is derived from `CAP.PREC.STAT.EFF_EW`.

### **15.4 Streaming Profiles & Prefetch Rules**

Streaming detection and prefetch behavior must be identical across domains:

* Hardware must treat `CAP.XMEM.STREAM_CTL` as the architectural default.
* Each domain may override via descriptor-local `XPHMG_MEMCTL.sprof_id`, but the fallback must be the CAP default.
* Prefetch depth, stride, and non-temporal policy **must not diverge** per domain.

Example:

* MTX tile loads using a profile must behave identically to RSV loads using the same profile.
* RT node prefetch granularity must reflect the same `SPROF` default when no override is given.

### **15.5 Memory Classes & Domain Semantics**

All memory-touching instructions must obey:

1. `CAP.XMEM.DOM`

   * Defines default coherence behavior (CPU / RT / MTX / HYB).
2. `CAP.XMEM.CLSMAP`

   * Defines the mapping of memory classes CL0–CL3 to cache ways/QoS.
3. `CAP.XMEM.LDS_BUDGET`

   * Soft per-class LDS allocation hints.

Domains **may not** introduce their own class or domain mapping rules.

Example:
If `CL0` is mapped to RT traversal, *all engines* using CL0 must share the same priority/way mask behavior.

### **15.6 LDS Behavior Consistency**

The LDS allocation and barrier semantics defined earlier in this document must be respected by all domains:

* `XLDS.ALLOC/FREE` must apply uniformly.
* `XLDS.BAR.*` must enforce visibility for all engines.
* Implementations must not redefine LDS alignment or tile layout rules per domain.

This ensures:

* MTX tiles, RT BVH nodes, RSV temporary blocks, and GFX shading intermediates all share the same LDS semantics.

### **15.7 Descriptor Integration (`XPHMG_MEMCTL`)**

Any domain using descriptors (NPU, RT optional path, GFX jobs, MTX submission) must treat the trailing `XPHMG_MEMCTL` structure as an override **only** for the fields it explicitly sets:

* `dom` (coherence domain override)
* `mclass` (explicit memory class, 0–3)
* `sprof_id` (streaming profile override)
* `hyb` (hybrid pipeline marker)
* `sec` (security domain/partition ID)

All unspecified fields must fall back to `CAP.XMEM.DESCPOL` or the default CAP.XMEM values.

### **15.8 Compression Consistency**

When inline compression is enabled (`CAP.XMEM.COMP_CTL`):

* All domains must use the same default mode/dictionary.
* Engines may accelerate compression differently internally, but the architectural result must be identical.
* If a domain does not support inline compression, it must ignore compressed hints and issue uncompressed accesses **while setting `COMP_ERR` in `CAP.XMEM.EVENTS`**.

### **15.9 Error & Event Reporting**

All domains must set `CAP.XMEM.EVENTS` for:

* OOB or misaligned accesses (`INV_MISS`, `BOUND_CHECK`)
* LDS exhaustion (`LDS_OOM`)
* Cache lock overflows (`LOCK_OVF`)
* Compression failures (`COMP_ERR`)
* Security violations (`SEC_VIOL`)
* Hybrid class fallback (`HYB_FALLBK`)

This ensures uniform debugging and telemetry across heterogeneous pipelines.

### **15.10 Deterministic Replay / Debug Requirements**

To enable deterministic replay and debugging:

* Debuggers must observe the *latched* XMEM state per instruction boundary.
* Trace tools must show XMEM/SVMEM state synchronized with CAP.PREC state.
* Engines must not expose partially updated XMEM configurations.


### **15.11 Summary (Normative)**

All domains must appear as if they execute under *one unified memory system*, defined entirely by CAP.XMEM, with the following guarantees:

* **Determinism:** no mid-instruction effects.
* **Cross-domain consistency:** same SVMEM, ZMODE, FOF, streaming, LDS, class mapping.
* **Descriptor coherence:** per-job overrides only modify the documented subset.
* **Correctness:** any deviation must raise a memory-event sticky.
* **Portability:** software pipelines (RT↔MTX↔RSV↔GFX↔NPU) remain valid on all XPHMG implementations.

---

## 16. Compatibility

* Precision and numeric control are **fully delegated to `XPHMG_PREC`**.
* Features missing from `xmem_cap` may be ignored without functional impact.
* Legacy `XMEM.DMA` / `XMEM.FENCE` opcodes are **deprecated** and reserved.
* All previously defined opcodes (`XLDS.*`, `XMEM.LDG/STG/PREF/CCTL`, `XSWZ.*`, `XMEM.DLOADC/DSTOREC`) **remain as-is**. The only change is the **source of configuration** (now CAP).

---

## 17 Normative statements

* **N-1.** At instruction decode, engines must observe the latest `CAP.XMEM.*` state; any engine-local mirrors must be coherent with CAP at the instruction boundary.
* **N-2.** If a feature bit in `CAP.XMEM.CAP` is clear, requests targeting that feature are ignored safely; architectural state reads return zero.
* **N-3.** Gather/scatter must honor RSV predication and `CAP.PREC.MODE.ZMODE`; with `FOF=1`, the first faulting element index must be written to `RSV.SVFAULTI` and the operation aborted.
* **N-4.** LDS allocation and streaming policies must enforce CAP budgets and classes on a best-effort basis, reporting overflows via `CAP.XMEM.EVENTS`.
* **N-5.** Descriptor fields override CAP defaults only when non-zero and supported; otherwise CAP defaults apply.

---
Designed for integration with `XPHMG_RSV`, `XPHMG_MTX`, `XPHMG_RT`, and `XPHMG_CAP`.

*Licensed under CC-BY-SA 4.0 — Open Specification maintained by Pedro H. M. Garcia.*
