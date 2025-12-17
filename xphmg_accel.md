# RISC-V Vendor Extension **XPHMG_ACCEL**

**Generic Descriptor-driven Accelerators / NPU**

- **Category:** Vendor Extension (`XPHMG_ACCEL`)
- **Version:** 0.1.0
- **Depends on:** `XPHMG_CAP`, `XPHMG_XMEM`
- **Author:** Pedro H. M. Garcia  
- **License:** CC-BY-SA 4.0 (open specification)

- **Author:** Pedro H. M. Garcia  
- **License:** CC-BY-SA 4.0 (open specification)

---

## 1. Scope & Goals (Informative)

`XPHMG_ACCEL` defines a **descriptor-driven interface** for integrating specialized accelerators into the XPHMG execution model.

Rather than introducing accelerator-specific numeric or memory control, ACCEL workloads inherit all architectural policy from `XPHMG_CAP` and `XPHMG_XMEM`.

Accelerators appear as additional execution domains that:

* Consume the same CAP state as other XPHMG engines,
* Use the same memory classes, coherence domains, and streaming profiles,
* Interact with the system exclusively through explicit descriptors and queues.

This model enables NPUs, DSPs, and custom offload engines to integrate cleanly into hybrid graphics and compute pipelines without fragmenting the architectural contract.

The key goals:

1. **Reuse global state**  
   All accelerator jobs *inherit* numeric policy from `CAP.PREC.*` and
   memory policy from `CAP.XMEM.*` (including SVMEM). No separate
   “precision CSRs” exist in ACCEL.

2. **Descriptor-first programming model**  
   Workloads are dispatched via **descriptor queues** and optional
   doorbell registers, not ad-hoc CSRs.

3. **Cross-domain consistency**  
   ACCEL engines must look like “just another domain” alongside RSV,
   MTX, RT, and GFX: same CAP, same XMEM semantics, same error/event
   reporting.

---

## 2. Dependencies & Architectural Position

### 2.1 Relationship to CAP and XMEM

`XPHMG_ACCEL` **does not define** its own precision or memory CSRs.
Instead, it is normatively bound to:

- `XPHMG_CAP` — for:
  - `CAP.PREC.MODE/ALT/STAT/EXC.*` (precision, formats, FP policy),
  - `CAP.FEAT*`, `CAP.TIER.*` (feature discovery & hints).:contentReference[oaicite:1]{index=1}  
- `XPHMG_XMEM` — for:
  - `CAP.XMEM.*` (L1/L2, LDS, classes, compression),
  - `CAP.XMEM.SVM*` (SVMEM indexed gather/scatter configuration).:contentReference[oaicite:2]{index=2}  

ACCEL units must **snapshot CAP and XMEM at descriptor granularity**:

> Any descriptor execution appears as if it ran under a single,
> well-defined `CAP.PREC.*` + `CAP.XMEM.*` state, observed at
> **descriptor start** and stable until completion.

### 2.2 CSR Map (high-level)

`XPHMG_ACCEL` deliberately **avoids** allocating CSRs in the
`0x7C0–0x7FF` XPHMG CAP window (already owned by `XPHMG_CAP` and
`XPHMG_RSV`).:contentReference[oaicite:3]{index=3}  

Instead, it defines:

- **Software-visible CSRs (suggested vendor window, e.g. 0x7A0+)**
  - Queue doorbells
  - Job/status registers
  - Optional performance counters

Specific numeric addresses remain **vendor-defined**, as long as:

- They do **not** conflict with the CAP/XMEM layout.
- Their semantics follow the normative rules in §5.

---

## 3. Descriptor Model

### 3.1 Base ACCEL descriptor layout

Implementations may provide multiple descriptor formats. This document
defines a **canonical “ACCEL base descriptor”** that software can rely
on for basic NPU-like workloads.

```c
struct XPHMG_ACCEL_DESC {
  // 0. Linkage / queue
  uint64_t next;        // Pointer to next descriptor (0 = end of chain)
  uint32_t flags;       // Common flags: START, END, FENCE_AFTER, INT_EN, etc.
  uint32_t dom;         // Coherence domain: DOM_CPU, DOM_RT, DOM_MTX, DOM_ALL

  // 1. Work geometry
  uint32_t workgroups[3]; // WG count in X/Y/Z (or tiles/batches)
  uint32_t local[3];      // Local size (optional; 0 = impl-defined default)

  // 2. Pointers (logical)
  uint64_t src0;        // Main input tensor / buffer
  uint64_t src1;        // Optional second input (weights, bias, etc.)
  uint64_t dst0;        // Main output
  uint64_t aux;         // Optional scratch / temporary

  // 3. Numeric / memory policy (overrides for CAP/XMEM)
  uint16_t prec_mode;   // 0 = inherit CAP.PREC.EFF_*, else small preset index
  uint16_t xmem_class;  // 0xFF = inherit CAP.XMEM.DOM/CLSMAP, else class id
  uint16_t sprof_id;    // Streaming profile 0–3 (0xFF = inherit CAP.XMEM.STREAM_CTL)
  uint16_t rsv0;

  // 4. Op encoding
  uint32_t op;          // Opcode / kernel id (e.g., CONV, MATMUL, SOFTMAX, etc.)
  uint32_t op_flags;    // Layout, activation, post-ops, etc.

  // 5. Dimensions / strides (tensor layout)
  uint32_t dims[4];     // N, C, H, W or generic D0..D3
  uint32_t stride_src0[4];
  uint32_t stride_src1[4];
  uint32_t stride_dst0[4];

  // 6. Vendor extension area (optional)
  uint64_t vendor[4];   // Reserved for vendor-specific fields
};
````

**Rules:**

1. Zero / sentinel values mean “**inherit from CAP/XMEM**”:

   * `prec_mode == 0` → use `CAP.PREC.STAT.EFF_*`.
   * `xmem_class == 0xFF` → use `CAP.XMEM.DOM` + `CAP.XMEM.CLSMAP`.
   * `sprof_id == 0xFF` → use `CAP.XMEM.STREAM_CTL` defaults.

2. Non-zero values act as **soft overrides**, but must remain compatible
   with CAP capabilities and XMEM configuration (see §5).

3. Descriptors are **located in virtual or physical memory** as defined
   by the platform ABI; ACCEL units must obey the same address/translation
   rules as other XPHMG domains.

### 3.2 Integration with `XPHMG_MEMCTL`

For platforms using the `XPHMG_MEMCTL` descriptor extension (defined in
XMEM), an ACCEL descriptor may either:

* Embed a full `struct XPHMG_MEMCTL` inline; or
* Contain an **index / pointer** into a MEMCTL table.

The recommended pattern:

```c
struct XPHMG_ACCEL_DESC_EXT {
  struct XPHMG_ACCEL_DESC base;
  struct XPHMG_MEMCTL    memctl; // Optional, may be omitted if all zeros
};
``` 

**Policy:**

* If `memctl` is **all zeros**, the engine must use CAP.XMEM defaults.
* If `memctl` fields are non-zero, they override the documented subset of
  `CAP.XMEM.*` for this job only (class, sprof, dom, etc.), in line with
  XMEM semantics.

---

## 4. Programming Model

A typical NPU/ACCEL workflow:

1. **Program CAP/XMEM once per phase**

   ```asm
   # Example: FP16 compute, FP32 accumulate, INT8 quantized path enabled
   # (see XPHMG_CAP for exact encoding)
   csrw 0x7D0, t0       # CAP.PREC.MODE
   csrw 0x7D1, t1       # CAP.PREC.ALT (optional FP8/INT4 modes)
   csrw 0x7E2, t2       # CAP.XMEM.L1CFG
   csrw 0x7E3, t3       # CAP.XMEM.CLSMAP
   csrw 0x7E7, t4       # CAP.XMEM.STREAM_CTL
   ```

2. **Build descriptors in memory**

   ```c
   XPHMG_ACCEL_DESC d = {0};

   d.workgroups[0] = tiles_x;
   d.workgroups[1] = tiles_y;
   d.src0 = input_ptr;
   d.src1 = weight_ptr;
   d.dst0 = output_ptr;

   d.op = ACCEL_OP_CONV2D;
   d.op_flags = ACCEL_FLAG_RELU | ACCEL_FLAG_NHWC;

   // Inherit CAP.PREC.* and CAP.XMEM.*:
   d.prec_mode  = 0;
   d.xmem_class = 0xFF;
   d.sprof_id   = 0xFF;
   ```

3. **Submit to queue**

   * Write descriptor pointer to a queue tail register.
   * Ring a **doorbell CSR** (implementation-defined).
   * Hardware consumes descriptors until `next==0` or queue empty.

4. **Wait for completion**

   * Poll a status CSR, or
   * Use interrupts based on `flags.INT_EN`, or
   * Use memory-mapped completion records.

---

## 5. Interoperability & CAP/XMEM Integration (Normative)

This section is the ACCEL-specific specialization of the global
interoperability rules already established for XPHMG domains.

### 5.1 Snapshot semantics

* At **descriptor fetch** time, the ACCEL engine must snapshot:

  * `CAP.PREC.STAT` (effective numeric policy),
  * `CAP.XMEM.*` and `CAP.XMEM.SVM*` (memory and gather/scatter policy).
* All operations for that descriptor are defined relative to that
  snapshot; mid-descriptor writes to CAP/XMEM take effect **only** on
  subsequent descriptors.

### 5.2 Precision / format rules

1. ACCEL ops must treat `CAP.PREC.STAT.EFF_*` as the **canonical**
   element type, width, accumulator width, and packing mode.

2. If a descriptor requests an unsupported `prec_mode` (e.g., mapping to
   FP8 when `CAP.PREC.FP8` indicates no support), the engine must:

   * Either:

     * Fallback to the nearest supported mode, and set
       `CAP.PREC.STAT.UNSUP_FMT`, or
   * Trap as *Illegal Instruction / Unsupported Descriptor* (vendor
     choice, but must be documented).

3. All FP exceptions raised by ACCEL ops must update
   `CAP.PREC.EXC.ST` stickies and honor `CAP.PREC.EXC.EN` + SAE policy,
   just like RSV/MTX/RT.

### 5.3 Memory rules

1. ACCEL loads/stores must obey `CAP.XMEM.DOM`, `CAP.XMEM.L1CFG`,
   `CAP.XMEM.CLSMAP`, `CAP.XMEM.LDS_BUDGET`, streaming profiles, and
   XMEM event semantics.

2. Indexed / indirect addressing used by ACCEL must deploy **exactly the
   same** SVMEM configuration (`CAP.XMEM.SVM*`) as RSV/MTX/GFX/RT:

   * Predication must respect `SVPMASK*` when ACCEL ops are driven by RSV
     masks (hybrid vector/NPU flows).
   * Zeroing vs merge for masked lanes must respect
     `CAP.PREC.MODE.ZMODE`.

3. ACCEL-specific memory failures (OOB, misalignment, LDS OOM, etc.)
   must set appropriate bits in `CAP.XMEM.EVENTS` for unified telemetry.

---

## 6. Error Handling & Debug

### 6.1 Descriptor status

Each accelerator queue must have a notionally simple **status word**
per descriptor, e.g.:

```c
struct XPHMG_ACCEL_STATUS {
  uint32_t done   : 1;  // 1 = completed
  uint32_t error  : 1;  // 1 = error, see code
  uint32_t code   : 6;  // error code (OOB, UNSUP_FMT, TIMEOUT, etc.)
  uint32_t rsv0   : 24;
  uint32_t cycles;      // optional: approximate cost
};
```

The ABI for where/how this status is exposed (MMIO vs memory) is
platform-defined, but:

* Errors caused by **numeric policy** must correlate with
  `CAP.PREC.EXC.ST` bits (NaNs, OF/UF, etc.).
* Errors caused by **memory policy** must correlate with
  `CAP.XMEM.EVENTS` bits.

### 6.2 Deterministic replay

To support deterministic replay:

1. Debuggers must be able to **dump CAP.PREC.STAT, CAP.XMEM.* and the
   descriptor contents** at job boundaries.
2. Engines must not expose **partially updated** CAP/XMEM or descriptor
   state midway through a job.
3. Vendors may implement shadow “mirror” registers for internal
   scheduling, but these are strictly micro-architectural and must
   always be coherent with CAP at descriptor boundaries.

---

## 7. Suggested Capability Bits (in CAP.FEAT / CAP.TIER)

While `XPHMG_ACCEL` itself does not define new CSRs, it recommends
exposing accelerator-related capabilities via `CAP.FEAT*` and
`CAP.TIER.*`, e.g.:

* `CAP.FEAT1.ACCEL_PRESENT` — generic “ACCEL/NPU present”.
* `CAP.FEAT1.ACCEL_DESC_CHAIN` — supports descriptor chaining via `next`.
* `CAP.FEAT1.ACCEL_MEMCTL` — supports embedded `XPHMG_MEMCTL` block.
* `CAP.TIER.ACCEL` (optional new CSR) — capability tier (e.g., MAC/s,
  max WG size, etc).

Exact bit assignments are vendor-specific but should be documented
aligned with the CAP spec.

---

## 8. Compatibility

- ACCEL uses `XPHMG_CAP` as the **sole** source of numeric policy.
- ACCEL uses `XPHMG_XMEM` as the **sole** source of architectural memory policy.
- Any vendor-specific ACCEL knobs should map to descriptor fields (e.g., `prec_mode`, `xmem_class`, `sprof_id`) or remain purely micro-architectural.

---
*Designed to integrate cleanly with `XPHMG_CAP`, `XPHMG_XMEM`, `XPHMG_RSV`,
`XPHMG_MTX`, `XPHMG_RT`, and `XPHMG_GFX` in XPHMG-NPU and Hybrid profiles.*
