# RISC-V Vendor Extension Family **XPHMG**
**Heterogeneous Graphics / Matrix / Ray / Vectors / NPU**

**Version:** 0.1.0  
**Author:** Pedro H. M. Garcia  
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Goals and Design Philosophy

**XPHMG** is a family of RISC-V vendor extensions that defines a unified **execution substrate** for modern graphics, geometry, and compute workloads.

Rather than exposing a fixed graphics pipeline or a driver-managed API, XPHMG provides a small set of **orthogonal architectural building blocks** that software composes explicitly:

* **CAP** — authoritative capability discovery and architectural numeric/memory policy,
* **XMEM** — explicit control over LDS, cache behavior, streaming, and cross-domain data residency,
* **RSV** — scalar-vector reinterpretation for compact and predictable vectorization,
* **MTX** — small matrix and tile compute primitives,
* **RT** — ray and geometry intersection primitives,
* **GFX** — texture, interpolation, shading, and visibility helpers,
* **ACCEL** — descriptor-driven offload for specialized accelerators.

XPHMG is **not** a graphics API, a driver model, or a fixed-function pipeline.
It does not perform implicit scheduling, state inference, or automatic memory promotion.
All observable behavior — numeric precision, masking, memory residency, coherence domain, and execution ordering — is **explicitly controlled by architectural state**.

---

### **1.1 Architectural Principles**

The design of XPHMG follows a small set of core principles:

1. **Single Source of Truth**
   All numeric behavior (precision, rounding, saturation, exceptions, quantization) and memory policy (classes, domains, streaming, gather/scatter) are owned by `XPHMG_CAP`.
   No extension redefines or overrides these rules.

2. **Explicit State, No Heuristics**
   Engines consume CAP and XMEM state as architecturally visible inputs.
   Implementations must not infer intent, reorder visible effects, or introduce hidden driver heuristics.

3. **Composable, Not Prescriptive**
   XPHMG defines primitives and conventions, not pipelines.
   Forward, deferred, tiled, hybrid, or compute-driven workflows are expressed by software composition, not by selecting a hardware mode.

4. **Deterministic Execution**
   Given an identical instruction stream and identical CAP/XMEM state, execution results are deterministic and debuggable.
   No implicit state transitions or background processing are permitted.

---

### **1.2 Execution Domains**

XPHMG extensions operate as **execution domains** that share a common architectural substrate:

* Domains may execute independently or cooperatively.
* Cross-domain dataflow is performed explicitly through XMEM-described regions.
* Logical ownership and priority are expressed via memory class and domain metadata, not by physical copies.

This model enables efficient pipelines such as MTX → RT → GFX or compute-only workflows without requiring implicit synchronization or opaque command processors.

---

### **1.3 Memory and Tiling Model**

XPHMG naturally supports **tile-based rendering and tiled compute** through:

* Explicit LDS/XMEM allocation,
* Tiered memory residency (Tier-0 / Tier-1 / higher tiers),
* Domain-controlled ownership and handoff,
* Canonical tile layouts (XDL) shared across engines.

Tiles are a **software-defined working set**, not an architectural object.
Their size, lifetime, and layout are fully controlled by software.

---

### **1.4 Scope of Version 0.1**

Version **0.1** defines the **first stable architectural baseline** of XPHMG.

It establishes:

* The CAP authority model,
* The unified XMEM memory semantics,
* The execution-domain framework,
* The instruction-level contracts of RSV, MTX, RT, GFX, and ACCEL.

Future revisions may extend capabilities or add optional features, but **must preserve the architectural guarantees defined in this baseline unless explicitly stated**.

---

## 2. Extension Map

| Extension          | Category         | Scope                                      | Depends on                 |
|--------------------|------------------|--------------------------------------------|----------------------------|
| `XPHMG_CAP`        | Vendor Extension | Capabilities, numeric & memory policy      | –                          |
| `XPHMG_XMEM`       | Vendor Extension | LDS/cache/streaming model + mem ISA        | `XPHMG_CAP` (XMEM block)   |
| `XPHMG_RSV`        | Vendor Extension | Simple-Vectors scalar-vector layer         | `XPHMG_CAP`, `XPHMG_XMEM`  |
| `XPHMG_RSV-Profiles` | Vendor Ext.    | RSV permute/compact/saturating profiles    | `XPHMG_RSV`, `XPHMG_CAP`   |
| `XPHMG_MTX`        | Vendor Extension | Small M×N matrix/tensor core               | `XPHMG_CAP`, `XPHMG_XMEM`  |
| `XPHMG_RT`         | Vendor Extension | Ray/AABB/Triangle, BVH, optional RT offload| `XPHMG_CAP`, `XPHMG_XMEM`  |
| `XPHMG_GFX`        | Vendor Extension | Graphics/texture pipeline ops              | `XPHMG_CAP`, `XPHMG_XMEM`, `XPHMG_RSV` (recommended) |
| `XPHMG_NPU`        | Descriptor Format| GEMM/CONV/POOL/ELTWISE descriptors         | `XPHMG_CAP`, `XPHMG_XMEM`  |

**Programming model rule:**  
At decode time, any XPHMG engine must observe:

1. The **effective precision state** from `CAP.PREC.MODE/ALT/STAT`.
2. The **effective memory policy** from `CAP.XMEM.*` (domains, classes, streaming, SVMEM).

Engine-local mirrors are allowed but non-architectural.

---

## 3. Core: Capabilities & Precision (`XPHMG_CAP`)

`XPHMG_CAP` provides:

1. **Capability discovery and hints** (RO, non-binding),
2. **Precision and numeric policy** (RW, architectural),
3. **Unified memory defaults** used by all domains (RW, architectural).

### 3.1 Capability & tier CSRs

- `CAP.ID`, `CAP.VERS`, `CAP.FLAGS` – vendor/version/global flags.  
- `CAP.FEAT0/1` – feature bits (bindless, RT presence, PMU, etc.).  
- `CAP.TIER.*`, `CAP.HINT.*` – register/LDS/RT tiers and alignment/BVH hints.

(Details in `XPHMG_CAP` spec.)

### 3.2 Precision & numeric policy

The **PREC block** defines global numeric behavior:

- `CAP.PREC.MODE` – base type & policy (PET/EW, ACCW, SAT, FP_RMODE, FTZ, Q/UNS, ZMODE, SAE, NaN policy).  
- `CAP.PREC.ALT` – alternate/experimental formats (FP8, INT4/INT2, BF16-Lite) + mixed/quant controls.  
- `CAP.PREC.STAT` – effective state & sticky flags (UNSUP_FMT, DOWNCAST_TAKEN, SAT_HIT, FTZ_HIT, IE_MASK).  
- `CAP.PREC.EXC.{EN,ST}` – FP exceptions enable and sticky bits.

All execution domains must:

- Snapshot **effective PET/EW/ACCW** and policy at instruction boundaries,
- Respect `ZMODE` for masked writeback (RSV masks, gather/scatter),
- Update stickies when saturation/FTZ/NaN events occur.

### 3.3 Memory & gather/scatter policy

`CAP.XMEM.*` holds architectural defaults for:

- Cache/LDS configuration (`L1CFG`, `L2CFG`, `CLS*`, `LDS_BUDGET`),
- Default domain (`DOM`),
- Streaming control and per-profile descriptors (`STREAM_CTL`, `SPROF0..3`),
- Inline compression defaults (`COMP_CTL`),
- Descriptor policy (`DESCPOL`),
- Gather/scatter control (`SVMBASE`, `SVMIDX`, `SVMSCL`, `SVMLEN`, `SVMMODE`),
- Sticky events (`EVENTS`).

Gather/scatter and stream behaviors in RSV, GFX, MTX, RT and NPU must **appear** as if they read these CSRs at decode.

---

## 4. Unified Memory / LDS / Streaming (`XPHMG_XMEM`)

`XPHMG_XMEM` defines:

- The **memory hierarchy model** (LDS, L1/L2 cache, XDMA, domains/classes),
- LDS allocation/barriers/atomics via `XLDS.*`,
- Cache and streaming loads/stores/prefetch via `XMEM.*`,
- Optional inline compression via `XMEM.DLOADC/DSTOREC`,
- Swizzle operations `XSWZ.*` (e.g., texture↔tensor layouts),
- Optional **engine-local mirror CSRs** (`XMIR.*`) for fast reprogramming.

Key points:

- Architectural source of truth is always `CAP.XMEM.*`.  
- Mirrors must be coherent with CAP at instruction boundaries.  
- `XPHMG_MEMCTL` is a reusable **descriptor tail** struct that carries:
  - `dom` (domain), `mclass` (memory class 0–3), `sprof_id` (streaming profile),
  - `hyb` (hybrid job), `sec` tag, hints for line size/QoS/LDS soft pinning.

This tail is used by RT and NPU descriptors and is intended to be shared by GFX/MTX submission paths.

---

## 5. Simple-Vectors (`XPHMG_RSV` + profiles)

`XPHMG_RSV` provides a **scalar-vector reinterpretation layer**:

- Prefixes (`svsetvl`, `svon.one`, `svon.blk`, `svend`) to enable vectorization over existing scalar ops.
- CSRs (`SVSTATE`, `SVSRCA/B`, `SVDST`, `SVPMASK*`, `SVFAULTI`) control VL, strides, masks, and fault index.
- `svon.fpctl` gives per-instruction override of FP rounding/SAE/ZMODE.

RSV’s behavior is explicitly tied to CAP:

- It interprets FP/INT ops according to `CAP.PREC.MODE/ALT/STAT`.
- It uses `CAP.XMEM.SVM*` for indexed gather/scatter.
- Masked lanes follow `ZMODE` (merge vs zeroing).
- FP exceptions are controlled by `CAP.PREC.EXC.{EN,ST}`.

### 5.1 Optional RSV profiles

`XPHMG_RSV-Profiles` defines:

- **XRSV-PRMT** – permutes, zip/unzip, table lookups analogous to SVE2/AVX-512.  
- **XRSV-CEXP** – compact/expand in registers + scatter-compact/gather-expand to memory.  
- **XRSVS** – saturating arithmetic, widening multiply/MAC, narrowing with saturation.

All of these:

- Reuse RSV’s mask & `ZMODE` semantics,
- Set `SAT_HIT` and related PREC stickies where appropriate,
- Use `CAP.XMEM.*` for their memory-touching forms (scatter/gather).

---

## 6. Matrix/Tensor Core (`XPHMG_MTX`)

`XPHMG_MTX` defines a compact **2×2..4×4** matrix/tensor execution path for:

- Small transforms (2×3 affine, 3×3 normals, 4×4 homogenous),
- Small blocks in denoisers/MLP layers,
- Mixed precision (e.g., FP8 input, FP16/FP32 accumulate).

It uses:

- **Precision policy** entirely from CAP (`PET/EW`, `ACCW`, `SAT`, `Q`, `PACK`),
- **Data movement** via XMEM (`XLDS.*`, `XMEM.*`), not via bespoke DMA.

Main CSRs:

- `MTXCFG`, `MTXSEL`, `MTX2/3/4`, `MTXBIAS` – banked storage and config,
- `MTXSTAT` – status and profile counters,
- `MTXCAP` – capabilities (max shape, bias support, etc.).

Main instructions (opcode `CUSTOM-2`):

- `MTX.LD` – convenience load of matrix/bias into selected bank,
- `MTX.MUL` – matrix×vector (optionally batched),
- `MTX.MAD` – matrix×vector + bias (if supported).

---

## 7. Ray/Geometry (`XPHMG_RT`)

`XPHMG_RT` provides:

- Scalar I-type **ray–AABB** (`RT.BBOX`) and **ray–triangle** (`RT.TRI`) primitives,
- A fixed **BVH node tile layout** (NODE4/NODE8) optimized for LDS/L1,
- Conventions for per-lane short stacks and queues in LDS,
- Optional descriptor-driven offload via `XSUBMIT.RT`.

It delegates:

- Numeric behavior (precision, rounding, FTZ, pack hints) to CAP,
- Memory behavior (LDS allocation, class, streaming profile) to XMEM + `XPHMG_MEMCTL`.

Optional offload descriptors (`XPHMG_RT_DESC`) contain:

- Pointers to BVH node heap, ray/hit/miss queues, triangle buffers,
- `opts` flags (culling, epsilon control, pack hints),
- A trailing `XPHMG_MEMCTL` block for domain/class/security hints.

---

## 8. Graphics / Texture (`XPHMG_GFX`)

`XPHMG_GFX` defines an instruction-level graphics/texture pipeline that:

- Shares the same register / memory / precision substrate as compute,
- Provides texture/tensor sampling, interpolation, shading/combine ops, cull/visibility helpers, basic RT prims, and fences,
- Relies on `XPHMG_RSV` for vectorization and `XPHMG_CAP/XMEM` for precision/memory behavior.

Implementations may:

- Map texture fetches and interpolation to LDS-first, class-aware memory paths,
- Use RSV profiles (permute/compact) to implement SOA/AOS layout shuffles and visibility compaction.

---

## 9. NPU / Tensor Offload (`XPHMG_NPU` descriptor)

`XPHMG_NPU` is a descriptor format (not a new opcode set) that:

- Describes GEMM/CONV/DEPTHWISE/POOL/ELTWISE/etc.,
- Uses a 64-byte header followed by an op-specific body,
- Reuses CAP’s PREC policy (PET/EW/ACCW) and XMEM’s LMEM/DRAM model,
- Supports post-ops (bias/add/activation) and quantization (zero-points, per-channel scales).

Submission uses a minimal CSR doorbell:

- `NPUCFG`, `NPUDESC` (doorbell), `NPUSTAT`, `NPUEVENT`, `NPUMAX`.

---

## 10. Unified Submission & `XPHMG_MEMCTL` (recommended)

For a coherent software stack, XPHMG recommends:

- A **common descriptor header** (size, version, op, flags, precision override, completion pointer, user tag),
- A **domain-specific body** (NPU/RT/GFX/MTX),
- A **trailing `XPHMG_MEMCTL` block** for memory/class/stream/security hints.

NPU (`XPHMG_NPU`) and RT (`XPHMG_RT_DESC`) already follow this pattern; GFX and MTX offload can be made to match.

---

## 11. Implementation Profiles

To ease adoption, vendors may implement subsets:

- **XPHMG-Base:** `XPHMG_CAP + XPHMG_XMEM`.  
- **XPHMG-RV-Vector:** Base + `XPHMG_RSV` (+ minimal RSV profiles).  
- **XPHMG-GFX:** Base + XMEM + RSV + GFX.  
- **XPHMG-RTU:** Base + XMEM + RT (+ MTX for denoisers).  
- **XPHMG-NPU:** Base + XMEM + NPU descriptors (no GFX/RT required).  
- **XPHMG-Hybrid:** Base + XMEM + RSV + MTX + RT (+ optional GFX/NPU).

Each profile should document:

- Which `CAP.PREC.*` formats are actually supported,
- Which XMEM capabilities are present (LDS, class partition, compression),
- Which RSV profiles and domains (GFX/RT/MTX/NPU) are implemented.

---

## 12. Optimization & Implementation Tips

**For software:**

1. **Program CAP once per phase.**  
   - Group work by precision/mode; avoid frequent writes to `CAP.PREC.*`.
2. **Use XMEM classes.**  
   - Dedicate CL0/CL1/CL2 to RT/MTX/HYB or GFX/compute, matching your pipeline.
3. **Use RSV for lane-local work.**  
   - Small vectors/structs: use RSV instead of heavy V extension.
   - Exploit RSV profiles for permutations, compaction, and saturating arithmetic.
4. **Use descriptors when possible.**  
   - Let NPU/RT units overlap compute and data movement.

**For hardware:**

1. **Latch CAP at instruction boundaries** (not mid-instruction).  
2. **Treat CAP.XMEM as architectural; mirrors are purely micro-architectural.**  
3. **Implement fault-only-first and SVFAULTI consistently** for vector/gather/scatter ops.  
4. **Favor FP32 accumulators** internally for FP8/FP16/BF16 unless area is extremely constrained.

---

## **13. Interoperability, Cross-Domain Guarantees & Implementation Guidelines**


This section provides a unified view of XPHMG behavior, prevents subtle cross-domain inconsistencies, and documents the architectural guarantees required for correct, scalable implement­ations.

The XPHMG family is designed so that **all execution domains** (RSV, MTX, RT, GFX, NPU) share the same precision, exception, and memory semantics.
To ensure forward compatibility and predictable behavior across heterogeneous units, the following interoperability rules apply:

### **13.1 Precision Interoperability (CAP → All Domains)**

All engines **must snapshot** the effective numeric state at the instruction boundary, as defined by `CAP.PREC.MODE`, `CAP.PREC.ALT`, and `CAP.PREC.STAT` :

* Effective element width (EW), primary element type (PET), accumulator width (ACCW), saturation mode, rounding mode, FTZ, NaN policy, ZMODE, SAE, and quantization settings **must be honored identically** by RSV, MTX, RT, GFX, and XMEM gather/scatter.
* Units must update the sticky flags (`SAT_HIT`, `FTZ_HIT`, `UNSUP_FMT`, `DOWNCAST_TAKEN`) consistently.
* Per-instruction overrides (e.g., RSV `svon.fpctl`) take effect *after* CAP defaults but revert automatically for the next instruction.

**Implementation requirement:**
Changes to CAP precision state must only take effect on the *next* instruction boundary. In-flight instructions must not partially observe new precision settings.

### **13.2 Unified Memory Semantics (CAP.XMEM → All Domains)**

All engines use `CAP.XMEM.*` as the *architectural source of truth* for:

* LDS allocation, budgets, class partitioning, and domain selection
* Cache behavior and streaming profiles
* Inline compression defaults
* Gather/scatter configuration (`SVMBASE`, `SVMIDX`, `SVMSCL`, `SVMLEN`, `SVMMODE`)
* Memory visibility and fence ordering
  - as defined in `XPHMG_XMEM`.

Engine-local mirrors (if implemented) **must be coherent** with CAP at every instruction boundary and may never diverge in architecturally visible behavior.

**Important:** This ensures that RSV gather/scatter, MTX tile loads, RT BVH traversal, GFX texture fetch, and NPU descriptors **all follow the same memory classes, domains, and streaming behavior**.

### **13.3 Cross-Domain Dataflow (RSV ↔ MTX ↔ RT ↔ GFX)**

XPHMG domains are intentionally designed so that **none supersedes any other**.
Each occupies a different performance/scale niche:

* **RSV (Simple-Vectors)** — lane-local vectorization, structural/permute ops, gather/scatter helpers.
  Ideal for *small vectors*, *SoA/AoS shuffles*, *prefix logic*, and *GFX/RT visibility compaction*.
  Defined in `XPHMG_RSV` .

* **MTX** — small-tile matrix MACs (2×2..4×4), mixed-precision inference, transforms.
  Ideal for *ML inference*, *small GEMM tiles*, *normals*, *skinning*, *denoisers*.
  Defined in `XPHMG_MTX` .

* **RT** — scalar per-lane ray–box/triangle tests and BVH traversal with LDS-first design.
  Ideal for *visibility*, *hybrid path tracing*, *G-buffer dependent passes*.
  Defined in `XPHMG_RT` .

* **GFX** — shading, texture fetch, raster helpers (optional in this spec).

Because all share CAP and XMEM:

* MTX can consume RSV-produced vectors without reformatting.
* RT can use MTX (e.g., transforms for instancing) without reprogramming precision.
* GFX can use RSV permutes to reorganize vertex or pixel data without moving memory.
* RSV mask semantics (`ZMODE`: merge vs zero) apply uniformly across domains.

This prevents “ISA feature cannibalization”: no domain becomes obsolete because each serves a distinct execution granularity.

### **13.4 Unified Tile Handoff Model**

When MTX produces a tile or composite tile into a Tier-0 `CLS_SHARED` region:

1. The tile is stored in canonical XDL layout.
2. Software or micro-scheduler changes ownership to `DOM_RSV`, `DOM_GFX` or `DOM_RT`.
3. The consumer engine reads the tile from XMEM directly, without copies or reinterpretation.

This mechanism replaces all forms of cross-pipeline register forwarding.

### **13.5 Primary Sources of Performance Bottlenecks**

Implementers should be aware of the following common pitfalls:

1. **Non-latched CAP state**
   Engines must latch CAP precision/memory state at decode.
   Mid-instruction CAP writes must **never** modify in-flight ops.

2. **Inconsistent gather/scatter semantics**
   All domains must respect SVMEM masks, `FOF`, zero/merge masked lanes, and index scale.

3. **LDS bank conflicts / soft budgets**
   Misconfigured class partitioning (CL0/CL1/CL2/CL3) may serialize RT and MTX.

4. **Poor integration between RSV permutes and MTX tile loads**
   RSV permutes should be used to get tiles into LDS in the right layout before MTX consumes them.

5. **Incorrect ZMODE semantics**
   Masked-lane writeback must follow `CAP.PREC.MODE.ZMODE`, not local unit defaults.

6. **Failure to propagate FP exception stickies**
   Stickies from MTX, RSV, scalar ALU, and RT operations must uniformly update
   `CAP.PREC.EXC.ST` and `CAP.PREC.STAT`.

### **13.6 Implementation Checklists**

#### **For hardware designers**

* Enforce boundary-based CAP latching.
* Create a single shared “effective precision” micro-structure that all domains read.
* Implement XMEM mirrors only if they simplify timing, and ensure instant sync on the next instruction boundary.
* Validate that RSV, MTX, RT, and GFX all consume SVMEM addressing identically.

#### **For driver/runtime authors**

* Program CAP once per phase, not per instruction.
* Configure XMEM classes based on pipeline (e.g., CL0=RT, CL1=MTX, CL2=HYB).
* Use RSV permutes to prepare data layouts for MTX and GFX.
* Use descriptor tails (`XPHMG_MEMCTL`) to steer domain/class/stream/security for NPU/RT/GFX job submissions.

---

## 14. Versioning and Change Management

- `XPHMG_CAP` exports a version (`CAP.VERS`) for the layout of CAP/XMEM.  
- Each domain spec (`RSV`, `RSV-Profiles`, `XMEM`, `MTX`, `RT`, `GFX`, `NPU`) has its own version and change log.  
- Forward-compatible implementations must treat unknown precision formats and XMEM capabilities as **“ignored safely”**, while setting `UNSUP_FMT` in `CAP.PREC.STAT` where appropriate.

---

# **Appendix A — Tile Rendering Model (Informative)**

## A.1 Scope and Intent

This appendix describes how **tile-based rendering** can be expressed naturally within the XPHMG execution model.

XPHMG **does not define a fixed graphics pipeline**, nor does it introduce an implicit “tile renderer.”
Instead, tile rendering is treated as a **software-defined execution pattern** composed of:

* Explicit memory placement (via `CAP.XMEM.*` and `XPHMG_MEMCTL`),
* Explicit execution domains (RSV / MTX / GFX / RT),
* Explicit synchronization and visibility rules.

This model applies to **modern rendering and compute workloads**, including forward, deferred, hybrid, and visibility-driven pipelines.

No additional instructions are required to support tile rendering.

---

## A.2 Definition of a Tile in XPHMG

In XPHMG, a **tile** is not an architectural object.
It is a **logical working set** defined by software and characterized by:

* Residency in **LDS / XMEM Tier-0 or Tier-1** memory,
* A bounded spatial extent (e.g., screen-space region, cluster of primitives, or visibility batch),
* Local reuse across multiple execution phases.

The size, shape, and contents of a tile are entirely software-controlled.

---

## A.3 Memory Placement and Residency

Tile rendering relies on predictable locality.
XPHMG provides this via explicit memory control.

### A.3.1 Recommended Memory Configuration

For tile workloads, software is expected to use:

* **Tier-0 (`T0`) or Tier-1 (`T1`) XMEM regions** for tile-local data,
* A dedicated **memory class** (e.g., `CL0`) for tile hot data,
* An appropriate **domain (`dom`)** matching the active execution engine.

Typical tile-resident data includes (non-exhaustive):

* Visibility lists,
* Depth or hierarchical Z buffers,
* G-buffer attributes,
* Intermediate shading results,
* Small tiles or blocks in canonical XDL layouts.

No implicit promotion or eviction is assumed.

---

## A.4 Tile Rendering Phases (Conceptual)

Tile rendering in XPHMG is expressed as a sequence of **explicit phases**.
These phases are **conceptual**, not architectural requirements.

### A.4.1 Tile Setup / Binning Phase

Purpose:

* Determine which primitives, fragments, or work-items belong to a tile.

Typical execution:

* RSV for compaction, prefix sums, index generation,
* MTX for transforms or small matrix operations,
* Writes to tile-resident lists or buffers in LDS/XMEM.

All data placement and ordering are explicit.

---

### A.4.2 Tile Shading Phase

Purpose:

* Perform depth testing, sampling, shading, and local composition.

Typical execution:

* GFX operations (`ZTEST`, sampling, interpolation, shading helpers),
* RSV for per-lane or per-fragment control,
* Optional MTX for small MLPs, denoisers, or transforms.

All reads and writes operate on tile-resident data.

Hierarchical Z or early visibility mechanisms are expressed explicitly using GFX and memory ordering primitives.

---

### A.4.3 Tile Resolve Phase

Purpose:

* Commit tile results to higher tiers of memory.

Typical execution:

* Explicit `XMEM.STG` or streaming stores,
* Optional compression or format conversion,
* Explicit fences to guarantee visibility.

No implicit resolve or flush is performed by hardware.

---

## A.5 Synchronization and Ordering

XPHMG requires **explicit synchronization** between tile phases.

Ordering guarantees are established using:

* LDS barriers (`XLDS.BAR`),
* Cache control and fences (`XMEM.CCTL`, `XFENCE.*`),
* Domain and class visibility rules from `CAP.XMEM.*`.

No execution phase implicitly synchronizes with another.

---

## A.6 Layout and Data Interoperability

Tile-local data may be stored using any layout expressible by the XPHMG Data Layout Model (XDL), including:

* Linear arrays,
* Canonical `XDL_TILE4x4_ROWMAJOR` tiles,
* Composite tiles,
* Engine-specific structured records.

When tiles are shared across execution domains, layouts must be interpreted consistently according to XDL rules.

---

## A.7 Determinism and Debug Visibility

Tile rendering in XPHMG is **fully deterministic** given:

* An identical instruction stream,
* Identical `CAP.PREC.*` and `CAP.XMEM.*` state,
* Identical descriptor parameters where used.

All state affecting execution is architecturally visible.
No hidden driver heuristics or implicit scheduling stages are permitted.

---

## A.8 Relationship to Immediate and Hybrid Rendering

The tile rendering model described here is **not exclusive**.

The same XPHMG implementation may support:

* Immediate-mode rendering,
* Tile-based rendering,
* Hybrid visibility or compute-driven pipelines,

without requiring ISA changes.

Tile rendering is an **execution strategy**, not a hardware mode.

---

## A.9 Summary

XPHMG supports tile rendering by design through:

* Explicit memory residency and classes,
* Explicit execution phases,
* Explicit synchronization,
* Explicit data layout rules.

This approach avoids implicit pipelines and opaque driver behavior while enabling efficient, modern tile-based rendering and compute workflows.

---

*This overview is non-normative; individual extension documents (`XPHMG_CAP`, `XPHMG_XMEM`, `XPHMG_RSV`, etc.) remain the normative source for instruction encodings and CSR bit layouts.*
