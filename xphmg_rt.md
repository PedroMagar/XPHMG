# RISC-V Vendor Extension **XPHMG_RT**
**Ray/Geometry Primitives, BVH Traversal Conventions, and Optional RT Offload**

**Category:** Vendor Extension (`XPHMG_RT`)  
**Version:** 0.1.1
**Author:** Pedro H. M. Garcia
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview (Informative)

This section provides a high-level description of the XPHMG_RT extension, its purpose, and its role in the XPHMG ISA family. It is informative and does not introduce additional architectural requirements beyond the normative sections that follow.

`XPHMG_RT` defines a minimal set of **ray–geometry intersection primitives** and traversal conventions for visibility-driven workloads.

The extension provides ray–AABB and ray–triangle tests, along with conventions for BVH layout, traversal stacks, and wavefront-style queues.

`XPHMG_RT` does not define a complete ray tracing pipeline.
It exposes only the architectural primitives required to build ray tracing, path tracing, and visibility queries explicitly in software.

All numeric behavior is inherited from `XPHMG_CAP`, and all memory residency, streaming, and queue placement are controlled via `XPHMG_XMEM`.

This design enables RT workloads to compose cleanly with MTX, RSV, and GFX domains without introducing implicit scheduling or fixed-function stages.

---

## 2. Scope, Non-goals, and Terminology

This section defines the intended scope of XPHMG_RT, states explicit non-goals for v0.1, and introduces terminology used throughout the specification.

Scope. XPHMG_RT specifies architectural ray-geometry intersection primitives, BVH node layouts, and software-visible conventions for traversal data structures and queues. It also defines how RT instructions integrate with precision control (`XPHMG_CAP.PREC.*`) and memory policy (`XPHMG_XMEM`).

Non-goals. This specification does not define a full ray tracing pipeline, fixed-function traversal hardware, BVH construction algorithms, host-side APIs, or shading languages. Scene management, scheduling policy, and traversal heuristics are software-defined.

Terminology. The following terms are used in this document:

* **Ray record:** per-lane structure containing ray origin, direction, and traversal interval (`tmin`, `tmax`). Additional fields are software-defined unless explicitly specified.
* **BVH node tile:** fixed-size node layout containing child AABBs and metadata, stored in LDS or cached memory for traversal.
* **LDS/LMEM:** local memory space managed under `XPHMG_XMEM`, used for low-latency storage of tiles, queues, and stacks.
* **Predicate:** per-lane condition state used for masking and control flow. RT consumes only effective predication from PMASK as defined in `xphmg.md`:
  `pred[i] = PMASK.bank[pbank_eff][i]`, where `pbank_eff` is selected by architectural prefix or instruction field and `pbank=0` denotes a virtual ALL-ONES bank. RT does not define its own masking or lane-enable state.
* **Queue descriptor:** ring buffer descriptor with `{head, tail, cap, rsv, base}` fields describing ray, hit, and miss queues.
* **Effective precision:** numeric format derived from `XPHMG_CAP.PREC.*` state that governs operand interpretation and result rounding.

---

## 3. Dependencies

This section summarizes external extension dependencies. Required items define the minimum architectural environment for correct execution; recommended and optional items define expected integration points without being mandatory.

- **Required:** `XPHMG_CAP v1.0` (`CAP.PREC.MODE`, `CAP.PREC.ALT`, `CAP.PREC.STAT`)
- **Recommended:** `XPHMG_XMEM` (LDS dynamic, class partition, streaming profiles)
- **Optional:** `XPHMG_RSV` (vector/wave control), `XPHMG_MTX` (instance transforms), `XPHMG_GFX` (shading path)

NOTE: This specification assumes compatible versions of the referenced extensions. Implementations claiming `XPHMG_RT` support MUST implement at least the required versions listed above. If a required dependency is absent or below the required version, `XPHMG_RT` is not supported and RT instructions must trap as illegal or unsupported features. Recommended and optional dependencies may be omitted without affecting architectural correctness, but their associated integration behavior is unavailable.

---

## 4. Design Goals (Informative)

This section states informative design goals that motivated the extension. These goals do not impose additional requirements beyond the normative sections.

- **Small ISA surface:** two primitives (AABB, Triangle) cover most RT/PT workloads.
- **Precision-neutral:** all numeric policy comes from CAP.
- **Cache/LDS-oriented traversal:** BVH nodes and short stacks live in LDS; caches and DMA are controlled by XMEM (streaming-friendly).
- **Hybrid-ready:** domain/class hints route traffic (scheduling via descriptors).

---

## 5. Architectural State and CSRs

This section defines the architectural state exposed by XPHMG_RT through CSRs. These registers provide configuration, capability discovery, and status reporting for RT instructions and optional offload. Numeric and memory policy are controlled by `XPHMG_CAP` (`CAP.PREC.*`, `CAP.XMEM.*`) and `XPHMG_XMEM` and are not overridden by these CSRs.

### 5.1 RT CSRs (0x7FA0–0x7FA9)

This subsection lists the CSRs used by XPHMG_RT. The table specifies address, access type, and a short description of each register's architectural role.

| Addr  | Name       | Access | Description |
|-------|------------|--------|-------------|
| 0x7FA0| `RTCFG`    | RW     | Global flags/hints: `EN`, `LMEM_ONLY`, `EPS_DFLT`, `PACK_HINT_DFLT`. |
| 0x7FA1| `RTSTAT`   | RO     | Status/counters: `BUSY`, `ERR`, `LAST_EC`, `bbox_issued`, `tri_issued` (impl-defined). |
| 0x7FA2| `RTCAP`    | RO     | Capabilities: `NODE4`, `NODE8`, `WATERTIGHT`, `PACK_HINT`, `EPS_CTL`, `DESCQ`. |
| 0x7FA3| `RTBASE`   | RW     | Optional scene/root table base (when host wants a global anchor). |
| 0x7FA4| `RTQ_RAY`  | RW     | Ray queue descriptor (LMEM ring) — see §6. |
| 0x7FA5| `RTQ_HIT`  | RW     | Hit queue descriptor (LMEM ring). |
| 0x7FA6| `RTQ_MISS` | RW     | Miss queue descriptor (LMEM ring). |
| 0x7FA7| `RTCONF2`  | RW     | Secondary config: epsilon scalar; implementation-defined hint bits (must not affect predication or CAP.PREC.MODE.ZMODE). |
| 0x7FA8| `RTCLSDEF` | RW     | Default class/QoS: `{DEF_CLASS(1:0), HYB_ENABLE, HYB_BORROW_QOS(2:0)}` (mirrors `xmem_descpol`). |
| 0x7FA9| `RTSPROF`  | RW     | Default streaming profile id (0..3) used when descriptor omits `sprof_id`. |

> **Note:** Numeric precision/rounding/FTZ/saturation are defined by `XPHMG_CAP` (`CAP.PREC.*`) and are **not** in RT CSRs; RT uses the CAP snapshot latched at the instruction boundary.

Unless stated otherwise, RT CSRs reset to 0, RO fields ignore writes, and reserved bits read as 0 and ignore writes. RTSTAT counters, if implemented, are read-only and reads have no side effects.

**RTCFG (0x7FA0) layout**

| Bits       | Name            | Access | Description |
|------------|-----------------|--------|-------------|
| bit 0      | `EN`            | RW     | Global enable hint for RT instructions; software should set to 1 before issuing RT operations. |
| bit 1      | `LMEM_ONLY`     | RW     | Hint to prefer LDS/LMEM-resident traversal data where possible. |
| bit 2      | `PACK_HINT_DFLT`| RW     | Default pack hint when instruction does not set `PACK_HINT`. |
| bits[15:8] | `EPS_DFLT`      | RW     | Default epsilon scalar encoding; interpretation matches `RTCONF2.EPS_SCALE`. |
| bits[XLEN-1:16] | reserved   | RAZ/WI | Reserved. |

**RTSTAT (0x7FA1) layout**

| Bits       | Name       | Access | Description |
|------------|------------|--------|-------------|
| bit 0      | `BUSY`     | RO     | 1 if RT unit has in-flight work; 0 otherwise. |
| bit 1      | `ERR`      | RO     | 1 if `LAST_EC` is non-zero. |
| bits[7:2]  | `LAST_EC`  | RO     | Last error code (see Section 5.3). |
| bits[XLEN-1:8] | impl-defined | RO | Optional counters (e.g., `bbox_issued`, `tri_issued`); if unimplemented, read as 0. |

**RTCAP (0x7FA2) layout**

| Bits       | Name        | Access | Description |
|------------|-------------|--------|-------------|
| bit 0      | `NODE4`     | RO     | BVHNode4 tile support. |
| bit 1      | `NODE8`     | RO     | BVHNode8 tile support. |
| bit 2      | `WATERTIGHT`| RO     | Watertight triangle intersection support. |
| bit 3      | `PACK_HINT` | RO     | Packed output hints supported. |
| bit 4      | `EPS_CTL`   | RO     | Epsilon scaling control via `RTCONF2` supported. |
| bit 5      | `DESCQ`     | RO     | Descriptor-driven offload supported. |
| bits[XLEN-1:6] | reserved | RAZ/WI | Reserved. |

**RTBASE (0x7FA3) layout**

| Bits       | Name    | Access | Description |
|------------|---------|--------|-------------|
| bits[XLEN-1:0] | `BASE` | RW | Optional base address for a scene/root table; alignment and interpretation are software-defined. |

**RTQ_RAY/RTQ_HIT/RTQ_MISS (0x7FA4-0x7FA6) layout**

| Bits       | Name    | Access | Description |
|------------|---------|--------|-------------|
| bits[XLEN-1:0] | `QDESC` | RW | Address of a `Queue` descriptor (Section 6.3). |

**RTCONF2 (0x7FA7) layout**

| Bits       | Name          | Access | Description |
|------------|---------------|--------|-------------|
| bits[7:0]  | `EPS_SCALE`   | RW     | Epsilon scalar encoding used when `EPS_CTL=1`. |
| bits[9:8]  | `IMPL_HINT`  | RW     | Implementation-defined hints; must not affect PMASK predication or CAP.PREC.MODE.ZMODE. Reads as 0 if unsupported. |
| bits[XLEN-1:10] | reserved | RAZ/WI | Reserved. |

**RTCLSDEF (0x7FA8) layout**

| Bits       | Name               | Access | Description |
|------------|--------------------|--------|-------------|
| bits[1:0]  | `DEF_CLASS`        | RW     | Default memory class (mirrors `xmem_descpol`). |
| bit 2      | `HYB_ENABLE`       | RW     | Default hybrid enable. |
| bits[5:3]  | `HYB_BORROW_QOS`   | RW     | Default hybrid borrow QoS. |
| bits[XLEN-1:6] | reserved       | RAZ/WI | Reserved. |

**RTSPROF (0x7FA9) layout**

| Bits       | Name       | Access | Description |
|------------|------------|--------|-------------|
| bits[1:0]  | `SPROF_ID` | RW     | Default streaming profile ID (0..3). |
| bits[XLEN-1:2] | reserved | RAZ/WI | Reserved. |

### 5.2 Capabilities and Feature Discovery (RTCAP)

This subsection defines the capability discovery model. Software must read `RTCAP` and select code paths that only use supported features; absent capabilities must not be assumed.

`RTCAP` bit suggestions:

* `NODE4`, `NODE8`
* `WATERTIGHT`
* `PACK_HINT`
* `EPS_CTL`
* `DESCQ` (descriptor queue/offload present)

If a feature is absent:

* assembler/runtime must not request it, or
* the core traps with **Unsupported Feature**.

Undefined bits read as zero and ignore writes. `RTCAP` is read-only and has no read-to-clear side effects.

### 5.3 Errors and Status (RTSTAT, error codes)

This subsection describes architectural error reporting for RT instructions and optional offload. Errors may be surfaced by traps and by status reporting in `RTSTAT.LAST_EC`; precision-related events are reported via `CAP.PREC.STAT`.

* **Illegal Instruction:** shape/feature unsupported.
* **Unsupported Feature:** requesting `NODE8`/`PACK_HINT`/`DESCQ` on non-capable hardware.
* `RTSTAT.LAST_EC` records the last error code.
* Precision downgrades, saturation or FTZ are reported via `CAP.PREC.STAT`.

`RTSTAT.LAST_EC` encodings:

| Value | Name                  | Meaning |
|-------|-----------------------|---------|
| 0     | `EC_NONE`             | No error recorded. |
| 1     | `EC_ILLEGAL_INSTR`    | Illegal instruction in RT domain. |
| 2     | `EC_UNSUPPORTED_FEAT` | Unsupported feature requested. |
| 3-63  | Reserved              | Reserved for future use. |

`LAST_EC` resets to 0. If multiple errors occur, implementations may record the most recent error. No architectural clear mechanism is defined beyond reset.

---

## 6. Data Structures and Memory Layouts

This section specifies the data layouts and memory conventions that are architecturally visible to RT instructions. These layouts enable software and hardware to interoperate; any additional fields or higher-level structures are software-defined unless explicitly specified.

### 6.1 BVH Node Tiles

This subsection defines the BVH node tile format consumed by RT traversal. Software is responsible for constructing tiles that match these layouts before invoking `RT.BBOX`.

To improve locality, BVH nodes are stored as **tiles** in LDS (or cached in L1$):

**Alignment:** 64B per node, 64B-aligned — **Endianness:** little-endian

```c
struct BVHNode4 { // 64 bytes total
  // Child 0..3 AABBs (format per CAP.PREC effective state; quantization optional)
  u32 c0_min_x, c0_min_y, c0_min_z;
  u32 c0_max_x, c0_max_y, c0_max_z;
  u32 c1_min_x, c1_min_y, c1_min_z;
  u32 c1_max_x, c1_max_y, c1_max_z;
  u32 c2_min_x, c2_min_y, c2_min_z;
  u32 c2_max_x, c2_max_y, c2_max_z;

  // Child metadata in last 16B
  u32 child0_idx;
  u32 child1_idx;
  u32 child2_idx;
  u32 child_flags; // [1:0]c0,[3:2]c1,[5:4]c2,[7:6]c3  01=node,10=leaf(TriRange)
};
```

**Leaf format:**

```c
struct TriRange {
  u32 first;    // first triangle index
  u32 count;    // number of triangles
  u64 base;     // triangle buffer base (LDS/L1$/DRAM)
};
```

Optional **BVHNode8** (128B) may be advertised by `RTCAP.NODE8`.

**BVHNode8 layout (128B)**

When `RTCAP.NODE8=1`, a BVHNode8 is defined as two consecutive BVHNode4 tiles:

```c
struct BVHNode8 { // 128 bytes total
  struct BVHNode4 lo; // children 0..3
  struct BVHNode4 hi; // children 4..7
};
```

Child indices and flags in `lo` apply to children 0..3; fields in `hi` apply to children 4..7. Both tiles share the same node index space and triangle ranges.

### 6.2 Ray Records

Ray records are per-lane structures stored in LDS or cached memory. RT instructions read the minimum fields listed below; additional fields are software-defined and are not interpreted by hardware.

Minimum fields:

```
origin.xyz, dir.xyz, tmin, tmax
```

Format/packing follow **effective precision** from `XPHMG_CAP.PREC.*`.

NOTE: Alignment, padding, and maximum ray record size follow `XPHMG_XMEM` allocation rules. The ray record base address must be aligned to at least the size of its element format (e.g., 2 bytes for FP16, 4 bytes for FP32). The minimum ray record size is eight consecutive elements in the order listed (`origin.xyz`, `dir.xyz`, `tmin`, `tmax`); additional fields may follow at higher offsets.

### 6.3 Wavefront Queues

Wavefront queues are ring buffers used for ray, hit, and miss traffic. The queue descriptor format matches the baseline definition; queue entry formats are software-defined unless specified by another extension.

Same layout as the baseline definition:

```c
struct Queue {
  u32 head, tail, cap, rsv;
  u64 base; // entries in LDS or DRAM
};
```

Queue bounds are defined by `cap`. Software must maintain `head` and `tail` as indices in the range `[0, cap)` and update them with modulo arithmetic. The queue is empty when `head == tail`; software must prevent overflow and underflow. Behavior is unspecified if `cap=0` or if `head`/`tail` are out of range.

### 6.4 Traversal Stack

The traversal stack is a small per-lane structure used by software-managed traversal. The entry format below is the minimum required by the conventions in this document; storage depth and spill policy are software-defined.

Small per-lane stack in LDS (spill/fill if needed). Entry:

```
{ node_idx, tnear, tfar }
```

NOTE: Stack depth, spill/fill behavior, and overflow handling are software-defined unless explicitly provided by an implementation.

---

## 7. Instruction Set (major opcode `CUSTOM-0`, suggested 0x0B)

This section defines the architectural encoding and semantics of the RT instructions. All RT instructions use the `CUSTOM-0` major opcode and the I-type 32-bit format. Each instruction operates per lane under effective PMASK predication (Section 8.2), may update predicate state for predicate-true lanes where defined, and optionally writes results to `rd` in the effective precision. The effective predicate bank is selected by architectural prefix or an instruction field when provided; otherwise `pbank=0` (virtual ALL-ONES) applies. Numeric behavior is governed by `XPHMG_CAP.PREC.*`, and memory access behavior is governed by `XPHMG_XMEM`.

All are **I-type 32-bit**:
`imm[11:0] | rs1 | funct3 | rd | opcode`

### 7.1 `RT.BBOX` — Ray–AABB (slab test)

`RT.BBOX` performs a per-lane ray-AABB slab test against the BVH node tile currently referenced by software. It produces a hit predicate and, when enabled, returns `tnear` and `tfar` for traversal ordering.

**Encoding**

```
flags[11:0] | rs1 | funct3=110 | rd | 00001011b
```

The immediate field encodes per-instruction options that control clamping, predicate-only behavior, and output packing hints.

**Flags (`imm`)**

* bit0 `T_CLAMP`    — clamp `t` into `[tmin,tmax]`
* bit1 `PRED_ONLY`  — update predicate only; do not write `rd`
* bit2 `PACK_HINT`  — request packed `(tnear,tfar)` (e.g., FP16) if supported
* bit3 `W_GUARD`    — if clip-space `w==0`, force miss
* bits[11:4] `RSV`  — zeros

Note: `PRED_ONLY` suppresses `rd` writeback only; predication and masked-lane suppression are unchanged.

**Operands**

* `rs1`: base of **ray record** (origin/dir/tmin/tmax)
* AABB: from current BVH node tile in LDS/L1$
* `rd` : if `PRED_ONLY=0`, returns `(hit_pred, tnear, tfar)` in the **effective** format

**Semantics**
Per-lane slab test; sets **predicate**; optionally writes `(tnear,tfar)`.
Predicate-false lanes are suppressed per Section 8.2.
Implementations **should** use FP32 internal math for robustness when inputs are ≤FP16/INT.

---

### 7.2 `RT.TRI` — Ray–Triangle (watertight)

`RT.TRI` performs a per-lane ray-triangle intersection using a watertight algorithm. It produces a hit predicate and, when enabled, returns the parametric hit distance and barycentric coordinates.

**Encoding**

```
flags[11:0] | rs1 | funct3=111 | rd | 00001011b
```

The immediate field encodes per-instruction options that control culling, predicate-only behavior, and output packing hints.

**Flags (`imm`)**

* bit0 `CULL_BACK`  — back-face culling
* bit1 `PRED_ONLY`  — predicate only
* bit2 `PACK_HINT`  — request packed `(t,u,v)` (e.g., FP16) if supported
* bit3 `EPS_CTL`    — use `RTCONF2` epsilon scaling
* bits[11:4] `RSV`

Note: `PRED_ONLY` suppresses `rd` writeback only; predication and masked-lane suppression are unchanged.

**Operands**

* `rs1`: base of **ray record**
* Triangle verts: fetched via `TriRange` (LDS/L1$/DRAM)
* `rd` : if `PRED_ONLY=0`, returns `(hit_pred, t, u, v)` in effective format

**Semantics**
Watertight Möller–Trumbore; updates predicate; optional `(t,u,v)` writeback.
Predicate-false lanes are suppressed per Section 8.2.
Internal accumulation may be FP32 regardless of input EW.

---

## 8. Cross-Domain Integration and Requirements (Normative)

This section defines cross-domain architectural requirements that ensure consistent behavior between RT and other XPHMG domains. It is normative and constrains implementations in areas of precision, masking, exception reporting, and memory behavior. It consolidates rules that apply uniformly across scalar, vector, and descriptor-driven RT paths.

To maintain the unified XPHMG execution model, every RT instruction — scalar or descriptor-driven — must follow **the same CAP/XMEM/RSV/MTX rules** used across all domains.

This section is **normative**.

---

### 8.1 Precision and Numeric Policy (CAP)

This subsection specifies how RT instructions inherit numeric state from `XPHMG_CAP` (`CAP.PREC.*`). The effective precision and exception controls are sampled at instruction decode and apply uniformly to the entire operation.

Per `XPHMG_CAP`, unsupported formats or precision combinations must be reported via `CAP.PREC.STAT.UNSUP_FMT`. RT must not reinterpret or bypass CAP-defined numeric policy.

RT instructions must consume the **effective precision**:

* PET/EW/ACCW
* FP rounding
* FTZ
* Signedness
* Quantization (Q/ZP/SCALE_SH)
* ZMODE
* SAE and FP exception enable bits

RT code **may not** override numeric behavior.

Every RT instruction (`RT.BBOX`, `RT.TRI`, `XSUBMIT.RT` micro-ops) must snapshot at decode:

* `CAP.PREC.MODE`
* `CAP.PREC.ALT`
* `CAP.PREC.STAT`
* `CAP.PREC.EXC.{EN,ST}`

RT derives from those:

* Effective PET/EW
* Effective accumulator width (ACCW)
* FP rounding mode
* FTZ behavior
* SAE state
* Quantization (Q/ZP/SCALE) rules
* Signedness
* Masking/ZMODE rules

#### **Forbidden:**

RT implementations **must not**:

* introduce custom epsilon schemes outside CAP control
* implement custom rounding
* reinterpret FP/INT formats
* define unique quantization behavior
* saturate/denormal-flush differently than CAP

#### **Allowed:**

RT may use **internal FP32 math for robustness**, but **output must still follow CAP-defined format**.

---

### 8.2 Predication Model and Masked Execution (PMASK)

RT consumes only effective predication as defined by the XPHMG predication model in `xphmg.md`:

    pred[i] = PMASK.bank[pbank_eff][i]

`pbank_eff` is selected by an existing architectural prefix or an instruction field where applicable. If no selector is present, `pbank=0` is implied and denotes a virtual ALL-ONES bank.

RT does not define alternate predicate sources, ray-enable bits, or implicit lane suppression mechanisms.

For any RT operation (inline or descriptor-driven), lanes with `pred[i]=0` are masked off and:

* MUST NOT execute traversal, intersection, or shading work.
* MUST NOT read or update ray state, payloads, hit records, or queues.
* MUST NOT raise exceptions or contribute to fault/termination events.
* MUST NOT produce side-effects.

Predicate-producing RT instructions update the selected PMASK bank only for predicate-true lanes; predicate-false lanes preserve prior predicate state. Termination, miss, and early-exit behavior is orthogonal to predication.

---

### 8.3 Memory Operations Under Predication (XMEM)

All RT memory interactions are governed by `XPHMG_XMEM`. For any RT stage that touches memory (including BVH traversal and node fetch, primitive/geometry fetch, texture sampling, buffer/image loads, atomics, ray payload storage, and hit/miss queue updates), predicate-false lanes MUST NOT issue memory accesses or produce memory side-effects. See `XPHMG_XMEM` for predication, SVMEM, and FOF details; fault-reporting sinks are defined by the consuming ISA domain.

---

### 8.4 Writeback Policy (CAP.PREC.MODE.ZMODE)

Writeback for predicate-false lanes is governed exclusively by `CAP.PREC.MODE.ZMODE`; RT defines no additional writeback or suppression policy.

| ZMODE | Behavior                                  |
| ----- | ----------------------------------------- |
| 0     | predicate-false lanes preserve destination |
| 1     | predicate-false lanes write architectural zero |

In RT contexts, "destination" includes any architecturally visible output produced by an RT operation, such as `rd` results, hit attributes, ray payload fields, and hit/miss record entries as defined by the consuming ISA domain. Store-like RT effects (payload/hit record/queue writes) remain suppressed for predicate-false lanes per Section 8.3; ZMODE applies to any architectural destinations that are written.

---

### 8.5 FP Exceptions & Sticky Flags

This subsection specifies how RT operations contribute to floating-point exception state and sticky flags. RT must follow the same exception and SAE rules used by RSV and MTX.

RT must update:

* `SAT_HIT`
* `FTZ_HIT`
* `DOWNCAST_TAKEN`
* `UNSUP_FMT`
* `EXC.ST.{NV,DZ,OF,UF,NX,SNAN_SEEN,QNAN_SEEN}`

#### Trap rules (aligned with RSV/MTX):

A trap occurs **only if**:

* The exception is enabled in `EXC.EN`
* AND `EFF_SAE = 0`

If SAE override applies (CAP or `svon.fpctl`):

* **Trap is suppressed**
* Sticky bits must still be set

The first-faulting lane index is reported via the architecturally visible mechanism defined by the consuming ISA domain.

---

### 8.6 Memory Behavior via XMEM

This subsection defines memory behavior for RT instructions and optional offload. All RT memory accesses (BVH tiles, triangle fetches, queues) are governed by `XPHMG_XMEM`; RT does not introduce an independent memory model.

RT consumes `CAP.XMEM.*` and SVMEM defaults as defined in `XPHMG_CAP` and `XPHMG_XMEM`. If a feature bit in `CAP.XMEM.CAP` is 0, the related behavior is disabled and the associated controls are RAZ/WI.

RT must use:

* LDS allocation rules
* Memory classes (CL0–CL3)
* Streaming profiles
* SVMEM for indexed accesses
* FOF mechanism
* Descriptors (`XPHMG_MEMCTL`)

RT must follow exactly:

* `CAP.XMEM.DOM` (coherence domain)
* Memory class mapping (CL0=RT typical)
* Streaming detector and profiles
* Compression (`COMP_CTL`)
* `DESCPOL` defaults

#### RT gather/scatter **must use SVMEM**, including:

* predication
* scaling rules
* FOF behavior
* ZMODE on masked lanes

#### BVH tile loads and triangle fetches:

RT must not implement custom DMA rules — only XMEM.

---

### 8.7 Descriptor Interop (`XPHMG_MEMCTL`)

This subsection specifies how descriptor-provided `XPHMG_MEMCTL` fields override CAP defaults for RT offload. Only explicitly set fields are used to override CAP state.

`XPHMG_MEMCTL` field definitions and default policies are specified by `XPHMG_XMEM` and `CAP.XMEM.DESCPOL`. RT descriptors must not introduce new memory policy beyond those definitions.

Only explicitly set fields override CAP:

* `dom`
* `mclass`
* `sprof_id`
* `hyb`
* `sec`

All others revert to CAP defaults.

No RT-specific memory policy is allowed.

---

### 8.8 Ray–Geometry Numeric Requirements

This subsection constrains the numeric outputs of `RT.BBOX` and `RT.TRI` to ensure consistent results across domains. It does not define new algorithms beyond the instruction semantics.

#### AABB tests:

* Must follow CAP numeric formats when writing `(tnear, tfar)`
* Internal FP32 is allowed, but results must be downcast according to CAP rules, triggering `DOWNCAST_TAKEN` if needed

#### Triangle tests:

* Must use watertight Möller–Trumbore
* Must obey `CULL_BACK` only as a geometric flag
* Barycentric `(u,v)` must follow PET/EW rules

NaN propagation must follow CAP rules.

---

### 8.9 Cross-Domain Dataflow Guarantees

This subsection defines interoperability guarantees between RT and other XPHMG domains. These statements constrain how data is interpreted across domains; they do not introduce new operations.

#### **RT ↔ RSV**

RT and RSV interoperate through masking, scheduling, and lane predicates. The following behaviors are required:

* RSV may generate rays or reorder rays
* Effective PMASK predication and `CAP.PREC.MODE.ZMODE` apply directly to RT

#### **RT ↔ MTX**

RT and MTX interoperate through shared numeric formats and dataflow of transformed rays. The following behaviors are required:

* MTX transforms of rays (instance transforms, skinning, camera matrices) must preserve numeric consistency
* RT may use MTX outputs directly with no reinterpretation

#### **RT ↔ GFX**

RT and GFX interoperate through shared memory formats and shader-visible outputs. The following behaviors are required:

* RT outputs (hits, UVs, depths) must be readable by GFX shaders
* Texture gathers using SVMEM must behave the same in both domains

#### **RT ↔ Scalar**

RT and scalar code interoperate through a consistent lane-0 behavior model. The following behaviors are required:

* Scalar fallback must equal lane-0 RSV behavior

---

### 8.10 CAP/XMEM State-Change Rules (“Apply semantics”)

This subsection defines how CAP and XMEM state changes are observed by RT instructions. State updates become effective at instruction boundaries, and RT must not observe partial updates.

RT must **not** partially observe precision/memory changes.

1. CAP writes (`APPLY0/1`) become effective only at next instruction boundary
2. RT instructions snapshot that state atomically
3. `svon.fpctl` does **not** modify global CAP state
4. Descriptor overrides are local to that dispatch

---

## 9. Optional Descriptor-Driven Offload (`XSUBMIT.RT`)

This section specifies the optional descriptor-driven offload path for RT. It is only available when `RTCAP.DESCQ=1`; otherwise, `XSUBMIT.RT` is unsupported and must trap as an unsupported feature.

When `RTCAP.DESCQ=1`, an implementation may expose an RT offload queue.
**Submission** uses host-side descriptors with trailing **`XPHMG_MEMCTL`** (see XMEM):

```c
struct XPHMG_RT_DESC {
  u64  bvh_base;     // BVH node heap (LDS/L1$/DRAM)
  u64  rayq_desc;    // queue of input rays
  u64  hitq_desc;    // queue of hits
  u64  missq_desc;   // queue of misses
  u64  tri_buf;      // triangle buffer base (optional if per-leaf)
  u32  opts;         // flags: CULL_BACK, EPS_CTL, PACK_HINT, etc.
  u32  rsv;

  struct XPHMG_MEMCTL memctl; // dom, mclass, sprof_id, hyb, sec, etc.
};
```

* **Doorbell:** `XSUBMIT.RT rd, rs1` (returns a ticket in `rd`).
* **Sync:** `XFENCE.WAIT ticket` / `XFENCE.SIGNAL`.
* **Behavior:** device may implement in-core micro-scheduler or dispatch to a dedicated RTU.

> This path is **optional**; scalar `RT.*` instructions remain normative. Descriptor-driven execution obeys the PMASK predication, XMEM memory, and ZMODE writeback rules in Section 8.

Ordering and completion: Submissions to a given RT offload queue are processed in FIFO order. The ticket returned by `XSUBMIT.RT` is monotonically increasing for that queue; completion of a ticket implies completion of that submission and all prior submissions to the same queue. `XFENCE.WAIT` for a ticket guarantees that descriptor-driven memory effects (including hit/miss queue updates) are visible according to `XPHMG_XMEM`. Ordering across distinct queues is unspecified.

---

## 10. Execution Model

This section provides an illustrative execution model for software-managed traversal. It is informative and does not impose mandatory ordering or microarchitectural structure beyond the architectural requirements stated elsewhere.

A typical wavefront loop:

1. **Ray gen / instancing:** (optional) `MTX.MUL` to transform rays to instance space.
2. **Traversal:** prefetch BVH tile → `RT.BBOX` to pick children → short-stack push/pop.
3. **Leaf test:** fetch triangles (stream profile), run `RT.TRI`.
4. **Queues:** append to `RTQ_HIT`/`RTQ_MISS`; compact waves periodically.
5. **Hybrid phases:** use `memctl.mclass` (CL0/CL1/CL2) and `hyb` for denoise/composite passes.

---

## 11. Programming Examples (Informative)

This section provides non-normative examples that illustrate typical usage of RT instructions and related state. Examples assume the referenced extensions are available and configured as shown.

### 11.1 AABB test with packed outputs

This example demonstrates configuring precision state and requesting packed `(tnear,tfar)` results using `PACK_HINT`.

```asm
# Precision: FP16 base, ACCW=FP32
li  t0, ((0b10<<22) | (0b011<<19) | (0b01<<17) | (1<<31))
csrw 0x7D0, t0     # CAP.PREC.MODE.APPLY0=1

# BVH tile staged in LDS; test and get (tnear,tfar) packed
RT.BBOX  imm=(1<<2) /*PACK_HINT*/, rs1=a0, rd=a1
bep  nz, %skip_hit  # branch on predicate false (assembler pseudonym)
```

### 11.2 Triangle test with back-face culling

This example demonstrates a triangle test with back-face culling enabled.

```asm
RT.TRI   imm=(1<<0) /*CULL_BACK*/, rs1=a0, rd=a1
```

### 11.3 Offload submission (optional path)

This example demonstrates the optional descriptor-driven offload submission path.

```asm
la   a0, rt_desc
xsubmit.rt  t0, a0
xfence.wait t0
```

---

*CC-BY-SA 4.0 — Open Specification maintained by Pedro H. M. Garcia.*
Designed to integrate with `XPHMG_CAP`, `XPHMG_XMEM`, `XPHMG_MTX`, and optional `RSV`.
