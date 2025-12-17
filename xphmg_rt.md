# RISC-V Vendor Extension **XPHMG_RT**
**Ray/Geometry Primitives, BVH Traversal Conventions, and Optional RT Offload**

**Category:** Vendor Extension (`XPHMG_RT`)  
**Version:** 0.1.0 
**Author:** Pedro H. M. Garcia  
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview (Informative)

`XPHMG_RT` defines a minimal set of **ray–geometry intersection primitives** and traversal conventions for visibility-driven workloads.

The extension provides ray–AABB and ray–triangle tests, along with conventions for BVH layout, traversal stacks, and wavefront-style queues.

`XPHMG_RT` does not define a complete ray tracing pipeline.
It exposes only the architectural primitives required to build ray tracing, path tracing, and visibility queries explicitly in software.

All numeric behavior is inherited from `XPHMG_CAP`, and all memory residency, streaming, and queue placement are controlled via `XPHMG_XMEM`.

This design enables RT workloads to compose cleanly with MTX, RSV, and GFX domains without introducing implicit scheduling or fixed-function stages.

---

## 2. Dependencies

- **Required:** `XPHMG_PREC v1.0` (`XPREC_MODE`, `XPREC_ALT`, `XPREC_STAT`)
- **Recommended:** `XPHMG_XMEM` (LDS dynamic, class partition, streaming profiles)
- **Optional:** `XPHMG_RSV` (vector/wave control), `XPHMG_MTX` (instance transforms), `XPHMG_GFX` (shading path)

---

## 3. Design Goals

- **Small ISA surface:** two primitives (AABB, Triangle) cover most RT/PT workloads.
- **Precision-neutral:** all numeric policy comes from CAP.
- **Cache/LDS-oriented traversal:** BVH nodes and short stacks live in LDS; caches and DMA are controlled by XMEM (streaming-friendly).
- **Hybrid-ready:** domain/class hints route traffic (scheduling via descriptors).

---

## 4. RT CSRs (0x7FA0–0x7FA9)

| Addr  | Name       | Access | Description |
|-------|------------|--------|-------------|
| 0x7FA0| `RTCFG`    | RW     | Global flags/hints: `EN`, `LMEM_ONLY`, `EPS_DFLT`, `PACK_HINT_DFLT`. |
| 0x7FA1| `RTSTAT`   | RO     | Status/counters: `BUSY`, `ERR`, `LAST_EC`, `bbox_issued`, `tri_issued` (impl-defined). |
| 0x7FA2| `RTCAP`    | RO     | Capabilities: `NODE4`, `NODE8`, `WATERTIGHT`, `PACK_HINT`, `EPS_CTL`, `DESCQ`. |
| 0x7FA3| `RTBASE`   | RW     | Optional scene/root table base (when host wants a global anchor). |
| 0x7FA4| `RTQ_RAY`  | RW     | Ray queue descriptor (LMEM ring) — see §6. |
| 0x7FA5| `RTQ_HIT`  | RW     | Hit queue descriptor (LMEM ring). |
| 0x7FA6| `RTQ_MISS` | RW     | Miss queue descriptor (LMEM ring). |
| 0x7FA7| `RTCONF2`  | RW     | Secondary config: epsilon scalar, mask behavior (impl-defined fields). |
| 0x7FA8| `RTCLSDEF` | RW     | Default class/QoS: `{DEF_CLASS(1:0), HYB_ENABLE, HYB_BORROW_QOS(2:0)}` (mirrors `xmem_descpol`). |
| 0x7FA9| `RTSPROF`  | RW     | Default streaming profile id (0..3) used when descriptor omits `sprof_id`. |

> **Note:** Numeric precision/rounding/FTZ/saturation are **not** in RT CSRs; they come from `XPHMG_PREC`.

---

## 5. BVH Node Tiles

To improve locality, BVH nodes are stored as **tiles** in LDS (or cached in L1$):

**Alignment:** 64B per node, 64B-aligned — **Endianness:** little-endian

```c
struct BVHNode4 { // 64 bytes total
  // Child 0..3 AABBs (format per XPHMG_PREC effective state; quantization optional)
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

---

## 6. Ray/Queue/Stack Conventions

### 6.1 Ray record (per lane, in LDS or cached)

Minimum fields:

```
origin.xyz, dir.xyz, tmin, tmax
```

Format/packing follow **effective precision** from `XPHMG_PREC`.

### 6.2 Wavefront queues (LMEM/LDS rings)

Same layout as the baseline definition:

```c
struct Queue {
  u32 head, tail, cap, rsv;
  u64 base; // entries in LDS or DRAM
};
```

### 6.3 Short traversal stack

Small per-lane stack in LDS (spill/fill if needed). Entry:

```
{ node_idx, tnear, tfar }
```

---

## 7. Instruction Set (major opcode `CUSTOM-0`, suggested 0x0B)

All are **I-type 32-bit**:
`imm[11:0] | rs1 | funct3 | rd | opcode`

### 7.1 `RT.BBOX` — Ray–AABB (slab test)

**Encoding**

```
flags[11:0] | rs1 | funct3=110 | rd | 00001011b
```

**Flags (`imm`)**

* bit0 `T_CLAMP`    — clamp `t` into `[tmin,tmax]`
* bit1 `PRED_ONLY`  — update predicate only; do not write `rd`
* bit2 `PACK_HINT`  — request packed `(tnear,tfar)` (e.g., FP16) if supported
* bit3 `W_GUARD`    — if clip-space `w==0`, force miss
* bits[11:4] `RSV`  — zeros

**Operands**

* `rs1`: base of **ray record** (origin/dir/tmin/tmax)
* AABB: from current BVH node tile in LDS/L1$
* `rd` : if `PRED_ONLY=0`, returns `(hit_pred, tnear, tfar)` in the **effective** format

**Semantics**
Per-lane slab test; sets **predicate**; optionally writes `(tnear,tfar)`.
Implementations **should** use FP32 internal math for robustness when inputs are ≤FP16/INT.

---

### 7.2 `RT.TRI` — Ray–Triangle (watertight)

**Encoding**

```
flags[11:0] | rs1 | funct3=111 | rd | 00001011b
```

**Flags (`imm`)**

* bit0 `CULL_BACK`  — back-face culling
* bit1 `PRED_ONLY`  — predicate only
* bit2 `PACK_HINT`  — request packed `(t,u,v)` (e.g., FP16) if supported
* bit3 `EPS_CTL`    — use `RTCONF2` epsilon scaling
* bits[11:4] `RSV`

**Operands**

* `rs1`: base of **ray record**
* Triangle verts: fetched via `TriRange` (LDS/L1$/DRAM)
* `rd` : if `PRED_ONLY=0`, returns `(hit_pred, t, u, v)` in effective format

**Semantics**
Watertight Möller–Trumbore; updates predicate; optional `(t,u,v)` writeback.
Internal accumulation may be FP32 regardless of input EW.

---

## 8. Precision & Policy (via `XPHMG_PREC`)

RT instructions must consume the **effective precision**:

* PET/EW/ACCW
* FP rounding
* FTZ
* Signedness
* Quantization (Q/ZP/SCALE_SH)
* ZMODE
* SAE and FP exception enable bits

RT code **may not** override numeric behavior.

---

## 9. XMEM Integration (LDS, classes, streaming)

RT must use:

* LDS allocation rules
* Memory classes (CL0–CL3)
* Streaming profiles
* SVMEM for indexed accesses
* FOF mechanism
* Descriptors (`XPHMG_MEMCTL`)

---

## 10. Optional Descriptor-Driven Offload (`XSUBMIT.RT`)

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

> This path is **optional**; scalar `RT.*` instructions remain normative.

---

## 11. Execution Model

A typical wavefront loop:

1. **Ray gen / instancing:** (optional) `MTX.MUL` to transform rays to instance space.
2. **Traversal:** prefetch BVH tile → `RT.BBOX` to pick children → short-stack push/pop.
3. **Leaf test:** fetch triangles (stream profile), run `RT.TRI`.
4. **Queues:** append to `RTQ_HIT`/`RTQ_MISS`; compact waves periodically.
5. **Hybrid phases:** use `memctl.mclass` (CL0/CL1/CL2) and `hyb` for denoise/composite passes.

---

## 12. Capabilities & Fallback

`RTCAP` bit suggestions:

* `NODE4`, `NODE8`
* `WATERTIGHT`
* `PACK_HINT`
* `EPS_CTL`
* `DESCQ` (descriptor queue/offload present)

If a feature is absent:

* assembler/runtime must not request it, or
* the core traps with **Unsupported Feature**.

---

## 13. Errors & Status

* **Illegal Instruction:** shape/feature unsupported.
* **Unsupported Feature:** requesting `NODE8`/`PACK_HINT`/`DESCQ` on non-capable hardware.
* `RTSTAT.LAST_EC` records the last error code.
* Precision downgrades, saturation or FTZ are reported via `XPREC_STAT`.

---

## 14. Programming Examples

### 14.1 AABB test with packed outputs

```asm
# Precision: FP16 base, ACCW=FP32
li  t0, ((0b10<<22) | (0b011<<19) | (0b01<<17) | (1<<31))
csrw 0x7E0, t0     # XPREC_MODE.APPLY0=1

# BVH tile staged in LDS; test and get (tnear,tfar) packed
RT.BBOX  imm=(1<<2) /*PACK_HINT*/, rs1=a0, rd=a1
bep  nz, %skip_hit  # branch on predicate false (assembler pseudonym)
```

### 14.2 Triangle test with back-face culling

```asm
RT.TRI   imm=(1<<0) /*CULL_BACK*/, rs1=a0, rd=a1
```

### 14.3 Offload submission (optional path)

```asm
la   a0, rt_desc
xsubmit.rt  t0, a0
xfence.wait t0
```

---

## **15. Cross-Domain Interoperability & Implementation Requirements (Normative)**

*(This is the new major section you requested.)*

To maintain the unified XPHMG execution model, every RT instruction — scalar or descriptor-driven — must follow **the same CAP/XMEM/RSV/MTX rules** used across all domains.

This section is **normative**.

---

### **15.1 CAP Precision → RT Numeric Behavior**

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

### **15.2 ZMODE (Zero vs Merge) for Masked Lanes**

If RT is invoked via RSV or uses predicate masks inside a wavefront:

| ZMODE | Behavior                                  |
| ----- | ----------------------------------------- |
| 0     | masked-off lanes preserve rd element      |
| 1     | masked-off lanes write architectural zero |

RT **must** use the same rules as RSV/MTX/GFX.

---

### **15.3 FP Exceptions & Sticky Flags**

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

The first faulting lane is recorded in `SVFAULTI` if invoked via RSV.

---

### **15.4 Unified Memory Behavior via XMEM**

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

### **15.5 Descriptor Interop (`XPHMG_MEMCTL`)**

Only explicitly set fields override CAP:

* `dom`
* `mclass`
* `sprof_id`
* `hyb`
* `sec`

All others revert to CAP defaults.

No RT-specific memory policy is allowed.

---

### **15.6 Ray–Geometry Numeric Requirements**

#### AABB tests:

* Must follow CAP numeric formats when writing `(tnear, tfar)`
* Internal FP32 is allowed, but results must be downcast according to CAP rules, triggering `DOWNCAST_TAKEN` if needed

#### Triangle tests:

* Must use watertight Möller–Trumbore
* Must obey `CULL_BACK` only as a geometric flag
* Barycentric `(u,v)` must follow PET/EW rules

NaN propagation must follow CAP rules.

---

### **15.7 Cross-Domain Dataflow Guarantees**

#### **RT ↔ RSV**

* RSV may generate rays or reorder rays
* RSV masking and ZMODE apply directly to RT

#### **RT ↔ MTX**

* MTX transforms of rays (instance transforms, skinning, camera matrices) must preserve numeric consistency
* RT may use MTX outputs directly with no reinterpretation

#### **RT ↔ GFX**

* RT outputs (hits, UVs, depths) must be readable by GFX shaders
* Texture gathers using SVMEM must behave the same in both domains

#### **RT ↔ Scalar**

* Scalar fallback must equal lane-0 RSV behavior

---

### **15.8 CAP/XMEM State-Change Rules (“Apply semantics”)**

RT must **not** partially observe precision/memory changes.

1. CAP writes (`APPLY0/1`) become effective only at next instruction boundary
2. RT instructions snapshot that state atomically
3. `svon.fpctl` does **not** modify global CAP state
4. Descriptor overrides are local to that dispatch

---


---

*CC-BY-SA 4.0 — Open Specification maintained by Pedro H. M. Garcia.*
Designed to integrate with `XPHMG_PREC`, `XPHMG_XMEM`, `XPHMG_MTX`, and optional `RSV`.
