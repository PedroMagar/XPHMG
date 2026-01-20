# RISC-V Vendor Extension **XPHMG_ACCEL**

**Generic Descriptor-driven Accelerators / NPU**

- **Category:** Vendor Extension (`XPHMG_ACCEL`)
- **Version:** 0.1.1
- **Depends on:** `XPHMG_CAP`, `XPHMG_XMEM`
- **Author:** Pedro H. M. Garcia
- **License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview (Informative)

`XPHMG_ACCEL` defines a descriptor-driven interface for integrating specialized accelerators into the XPHMG execution model. ACCEL engines consume in-memory descriptors and optional doorbell-style CSRs. Numeric and memory control remain in `XPHMG_CAP` and `XPHMG_XMEM`.

### 1.1 Conceptual Model

- ACCEL engines are execution domains alongside RSV, MTX, RT, and GFX. They use the same CAP precision state and XMEM memory policy and present comparable error and event reporting.
- Software constructs descriptor queues in memory and notifies ACCEL via implementation-defined doorbells. Hardware consumes descriptors until a chain terminates. No ad-hoc per-job CSRs are introduced.
- Numeric and memory policy are inherited from CAP/XMEM. Per-descriptor fields provide soft overrides only where explicitly allowed.

### 1.2 Architectural Guarantees and Invariants

- ACCEL does not define independent precision or memory CSRs. Numeric and memory behavior is governed by `CAP.PREC.*` and `CAP.XMEM.*` as observed at descriptor boundaries.
- ACCEL must appear as another domain: same CAP precision semantics (including exception stickies and SAE), same XMEM coherence/classing/streaming model, and unified error/event reporting via CAP/XMEM-defined surfaces.
- Descriptor execution is coherent with the architectural state captured at descriptor start. Writes to CAP/XMEM during execution affect only subsequent descriptors.

### 1.3 Interaction with CAP / XMEM / Other Extensions

- CAP provides precision capabilities, effective element types, and FP exception behavior used by ACCEL operations.
- XMEM provides memory domains, classing, LDS budgeting, and streaming controls used by ACCEL. ACCEL uses the same SVMEM configuration for indexed access, and predicated access consumes effective PMASK-based predication per the XPHMG predication model (see `xphmg.md`).
- ACCEL interoperates with other XPHMG domains without adding new coherence or exception domains. Capability discovery uses CAP.FEAT* and CAP.TIER.* (see Section 10).

### 1.4 Undefined / Reserved Behavior

- CSR addresses for doorbells/status and descriptor opcode/flag encodings are vendor-defined. Unspecified descriptor fields are reserved; software must not rely on their behavior.
- Virtualization, interrupt routing, and security isolation details are defined in later sections. Behaviors not covered are undefined in this revision.

### 1.5 Notes for Implementers (Informative)

- Align ACCEL doorbell and queue semantics with platform conventions used by other XPHMG domains.
- Changing CAP/XMEM between descriptors is the architectural way to alter numeric or memory policy. Per-descriptor overrides are advisory and must remain compatible with inherited CAP/XMEM configuration.

NOTE: The conceptual model and interaction with other XPHMG domains are covered in Sections 1 and 3; no separate architectural overview section is defined in this revision.

---

## 2. Conventions and Terminology (Informative)

- Numeric notation: Hexadecimal uses the `0x` prefix. Bit ranges use `msb:lsb` per the RISC-V style.
- Endianness: Descriptor fields and CSRs use platform endianness. For little-endian systems (default RISC-V profiles), multi-byte fields are little-endian unless a descriptor format states otherwise.
- Alignment: Descriptor structures are naturally aligned to the width of their largest member unless stated otherwise. Misaligned descriptor access is implementation-defined and may fault per platform ABI.
- Addressing: Descriptor pointers, data buffers, and optional MEMCTL blocks use the same virtual/physical addressing and translation rules as other XPHMG domains (see XMEM).
- Terminology:
  - ACCEL domain: Hardware execution domain that consumes ACCEL descriptors and inherits CAP/XMEM policy.
  - Descriptor queue: Contiguous or linked list of descriptors in system memory, consumed by ACCEL.
  - Doorbell: CSR write that notifies ACCEL hardware of new descriptors (address/encoding is vendor-defined).
  - Soft override: Descriptor field that may override inherited CAP/XMEM policy when permitted, subject to capability checks.
  - Vendor extension area: Reserved per-descriptor space for implementation-specific fields; software must not depend on its layout unless specified by the vendor.

NOTE: This section is informative and sets shared assumptions used by later normative sections.

---

## 3. Dependencies and Architectural Position (Normative Overview)

ACCEL is bound to existing XPHMG components and does not introduce independent precision or memory control state.

### 3.1 Relationship to CAP and XMEM

`XPHMG_ACCEL` does not define its own precision or memory CSRs. It is normatively bound to:

- `XPHMG_CAP` - for:
  - `CAP.PREC.MODE/ALT/STAT/EXC.*` (precision, formats, FP policy),
  - `CAP.FEAT*`, `CAP.TIER.*` (feature discovery and hints)
- `XPHMG_XMEM` - for:
  - `CAP.XMEM.*` (L1/L2, LDS, classes, compression),
  - `CAP.XMEM.SVM*` (SVMEM indexed gather/scatter configuration)

ACCEL units must snapshot CAP and XMEM at descriptor granularity:

> Any descriptor execution appears as if it ran under a single, well-defined `CAP.PREC.*` + `CAP.XMEM.*` state, observed at descriptor start and stable until completion.

### 3.2 CSR Window and Ownership

- ACCEL avoids allocating CSRs in the `0x7C0-0x7FF` XPHMG CAP window, which is reserved for `XPHMG_CAP` and `XPHMG_RSV`. Software must not expect ACCEL-visible CSRs in this range.
- Vendor-defined CSR windows for doorbells/status are permitted (e.g., `0x7A0+`) if they do not conflict with CAP/XMEM ownership and follow the architectural semantics in later sections.

### 3.3 Architectural Position Among XPHMG Domains

- ACCEL is an execution domain alongside RSV, MTX, RT, and GFX. It does not redefine coherence, precision, or memory models; it reuses CAP/XMEM for those purposes.
- CAP.FEAT* / CAP.TIER.* bits are the discovery and tiering surfaces for ACCEL capabilities (see Section 10). No separate ACCEL discovery CSRs are introduced.

NOTE: Virtualization, privilege interactions, and reset behavior for ACCEL CSRs follow platform conventions and are specified in later sections. Behaviors not yet covered remain to be clarified.

---

## 4. Programmer's Model (Normative)

- Work submission: Software constructs one or more descriptor queues in memory. A queue is either a contiguous ring or a linked chain via `next` pointers. Queue layout, ownership, and protection are platform-defined but must follow the addressing rules in Section 2 and the policy snapshot rules in Section 7.
- Notification (doorbells): Submission is initiated by writing a vendor-defined doorbell CSR with the descriptor (or queue) pointer. The doorbell CSR address and encoding are implementation-defined but must not conflict with CAP/XMEM CSR windows (see Section 5.1).
- Execution model: Hardware consumes descriptors in program order within a queue until termination (e.g., `next==0` or queue-empty). Implementations may prefetch or deschedule descriptors internally but must preserve observable ordering as defined by Section 7.
- Completion: Completion is signaled via one or more of: (a) status CSR updates, (b) memory-resident completion records, or (c) interrupts when enabled by descriptor flags. The mechanism is platform-defined and must allow correlation to descriptor identity.
- Privilege and isolation: Access to ACCEL doorbells, status, and queues follows platform privilege and protection mechanisms (e.g., S/U-mode access control, PMP/IOPMP for memory-resident queues). ACCEL does not define new privilege levels.
- Virtualization: Virtualized systems must provide a consistent view of CAP/XMEM state per guest and ensure that ACCEL descriptors observe the guest's CAP/XMEM snapshot at descriptor start. Interrupt routing and queue ownership in virtualized contexts are platform-defined.

NOTE: Implementation-specific optimizations (e.g., batching doorbells, internal work-stealing) are permitted if architectural ordering and visibility rules are maintained. Completion and error reporting details are defined in Sections 8 and 9.

---

## 5. Architectural State and CSRs (Normative)

ACCEL exposes a minimal CSR surface for submitting work and observing descriptor processing. Numeric and memory policy remain in CAP/XMEM; ACCEL CSRs must not re-specify or override those controls.

### 5.1 CSR Map (high-level)

`XPHMG_ACCEL` deliberately avoids allocating CSRs in the `0x7C0-0x7FF` XPHMG CAP window (already owned by `XPHMG_CAP` and `XPHMG_RSV`).

Instead, it defines:

- Software-visible CSRs (suggested vendor window, e.g. 0x7A0+)
  - Queue doorbells
  - Job/status registers
  - Optional performance counters

Specific numeric addresses remain vendor-defined, as long as:

- They do not conflict with the CAP/XMEM layout.
- Their semantics follow the normative rules in Section 8.

### 5.2 Required CSR Classes (functional equivalence)

- Doorbell CSR(s): Write-only notification supplying a descriptor or queue pointer (or identifier) to ACCEL hardware.
- Status/Job CSR(s): Readable state sufficient to observe queue/busy state and last completion/error code; must correlate with the completion/error mechanisms in Sections 8 and 9.
- Interrupt control (if interrupts are supported): CSRs to enable, mask, and acknowledge ACCEL interrupts; may be vendor-defined or reuse platform interrupt CSRs.

NOTE: Vendors may combine these classes into a single CSR block if functional behavior is preserved.

### 5.3 Optional CSR Classes

- Performance counters: Optional read-only CSRs for ACCEL performance telemetry (e.g., completed descriptors, cycles). Presence should be discoverable via CAP.FEAT* or vendor documentation.
- Debug/trace aids: Optional CSRs (e.g., last descriptor pointer, mirror registers) for observability; these must not alter architectural behavior.

### 5.4 Reset and Initial Conditions

- ACCEL CSRs (doorbell, status, optional counters) must reset to architecturally neutral values (e.g., queues idle, interrupts disabled). CAP/XMEM reset semantics are defined in their respective specifications.
- Software must program CAP/XMEM before first descriptor submission. ACCEL assumes the CAP/XMEM reset state until explicitly changed.

### 5.5 Privilege and Access

- Access to ACCEL CSRs follows platform privilege and protection rules (e.g., M/S/U visibility, delegation). ACCEL does not introduce new privilege levels.
- Memory-resident queues and completion buffers must be protected by standard platform mechanisms (e.g., PMP/IOPMP). ACCEL hardware must honor these protections when fetching descriptors or writing completions.

### 5.6 Widths and Access Size

- ACCEL CSRs are word-aligned and use standard RISC-V CSR access widths (XLEN). Wider values (e.g., 64-bit pointers on RV32) may be exposed via paired CSRs consistent with platform conventions.

NOTE: Exact field layouts and encodings for vendor-defined CSRs are outside this section. They must implement the functional behavior described above without conflicting with CAP/XMEM ownership.

---

## 6. Descriptor Formats (Normative)

This section defines the canonical ACCEL base descriptor and its normative field semantics. Vendors may define additional descriptor formats, but software must be able to rely on the base descriptor for functional portability. Alignment and endianness follow Section 2.

### 6.1 Base ACCEL descriptor layout

Implementations may provide multiple descriptor formats. This document defines a canonical "ACCEL base descriptor" that software can rely on for basic NPU-like workloads.

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
  uint16_t sprof_id;    // Streaming profile 0-3 (0xFF = inherit CAP.XMEM.STREAM_CTL)
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
```

**Rules:**

1. Zero / sentinel values mean inherit from CAP/XMEM:
   - `prec_mode == 0` -> use `CAP.PREC.STAT.EFF_*`.
   - `xmem_class == 0xFF` -> use `CAP.XMEM.DOM` + `CAP.XMEM.CLSMAP`.
   - `sprof_id == 0xFF` -> use `CAP.XMEM.STREAM_CTL` defaults.
2. Non-zero values act as soft overrides and must remain compatible with CAP capabilities and XMEM configuration (see Section 7).
3. Descriptors are located in virtual or physical memory as defined by the platform ABI. ACCEL units must obey the same address/translation rules as other XPHMG domains.

**Field semantics:**

- `next`: Pointer to the next descriptor; `next == 0` terminates a chain. Alignment follows the natural alignment of the descriptor unless the platform specifies stricter alignment.
- `flags`: Bit allocation is vendor-defined. Reserved bits must be zero. Flags must not redefine numeric or memory policy outside the descriptor fields and CAP/XMEM.
- `dom`: Selects the coherence/memory domain defined by XMEM (e.g., CPU/RT/MTX/ALL). Must be consistent with the platform domain map.
- `workgroups` / `local`: Work geometry parameters. Zero values in `local` imply implementation-defined defaults.
- `src0/src1/dst0/aux`: Logical addresses interpreted under the same translation and protection model as other XPHMG domains. Software must ensure these regions are valid and accessible per the memory rules in Section 7.
- `prec_mode` / `xmem_class` / `sprof_id`: Soft overrides of CAP/XMEM policy when non-sentinel. Must not exceed capabilities reported via CAP.FEAT* / CAP.TIER.*.
- `op` / `op_flags`: Operation identifier and modifier flags. Encodings are vendor-defined. Unsupported combinations must follow the error handling rules in Section 8.
- `dims` / `stride_*`: Tensor/layout description interpreted by the selected `op`. Zero values are interpreted per vendor definition or imply defaults; software must not assume implicit broadcasting unless defined by the operation.
- `vendor[4]`: Reserved for vendor-specific use. Software conforming to this specification must treat these fields as opaque and set them to zero unless targeting a vendor-defined contract.

NOTE: Descriptor contents are read-only to hardware during execution. Software must not modify an in-flight descriptor unless the platform ABI explicitly allows it.

### 6.2 Integration with `XPHMG_MEMCTL`

For platforms using the `XPHMG_MEMCTL` descriptor extension (defined in XMEM), an ACCEL descriptor may either:

- Embed a full `struct XPHMG_MEMCTL` inline; or
- Contain an index or pointer into a MEMCTL table.

The recommended pattern:

```c
struct XPHMG_ACCEL_DESC_EXT {
  struct XPHMG_ACCEL_DESC base;
  struct XPHMG_MEMCTL    memctl; // Optional, may be omitted if all zeros
};
```

**Policy:**

- If `memctl` is all zeros, the engine must use CAP.XMEM defaults.
- If `memctl` fields are non-zero, they override the documented subset of `CAP.XMEM.*` for this job only (class, sprof, dom, etc.), consistent with XMEM semantics.

NOTE: If both per-descriptor policy fields (e.g., `xmem_class`, `sprof_id`, `dom`) and a non-zero `memctl` are present, precedence is vendor-defined in v0.1 and must be documented by the vendor.

---

## 7. Operational Semantics (Normative)

This section defines how ACCEL interprets descriptors relative to CAP/XMEM state and executes memory and numeric operations while preserving architectural ordering and visibility.

### 7.1 Descriptor lifecycle and snapshot

- At descriptor fetch, the ACCEL engine must snapshot:
  - `CAP.PREC.STAT` (effective numeric policy),
  - `CAP.XMEM.*` and `CAP.XMEM.SVM*` (memory and gather/scatter policy).
- Operations for that descriptor are defined relative to that snapshot. Writes to CAP/XMEM during execution take effect only on subsequent descriptors.
- Hardware may prefetch descriptors speculatively, but visible execution must reflect the snapshot of the descriptor actually retired. Speculation must not expose incorrect CAP/XMEM observations to software.

### 7.2 Ordering and visibility

- Within a queue, descriptors execute in program order for architectural visibility of memory side-effects and completion. Implementations may overlap execution internally if they preserve this ordering at the memory model level.
- Descriptor memory operations must obey the ordering rules implied by XMEM domains/classes. Flags in `flags` may further constrain ordering if defined by the vendor.
- Completion (CSR update, interrupt, or completion record) must not be observed before all descriptor memory side-effects are visible under the selected XMEM domain/class semantics.

### 7.3 Precision / format rules

1. ACCEL operations must treat `CAP.PREC.STAT.EFF_*` as the canonical element type, width, accumulator width, and packing mode.
2. If a descriptor requests an unsupported `prec_mode` (e.g., mapping to FP8 when `CAP.PREC.FP8` indicates no support), the engine must either:
   - Fall back to the nearest supported mode and set `CAP.PREC.STAT.UNSUP_FMT`; or
   - Trap as Illegal Instruction / Unsupported Descriptor (vendor choice, but must be documented).
3. FP exceptions raised by ACCEL operations must update `CAP.PREC.EXC.ST` stickies and honor `CAP.PREC.EXC.EN` and SAE policy, consistent with RSV/MTX/RT.

### 7.4 Predication and masked execution

ACCEL consumes only effective predication as defined by the XPHMG predication model (`xphmg.md`). The only architecturally defined predicate source is PMASK:

```
pred[i] = PMASK.bank[pbank_eff][i]
```

where `pbank_eff` is selected by an existing architectural prefix or an instruction field (where applicable in ACCEL encodings). If no selector is present, `pbank = 0` is the default and denotes a virtual ALL-ONES bank.

For any predicated ACCEL operation:

1. Lanes/tiles/threads with `pred[i] = 0` MUST NOT execute and MUST NOT read operands.
2. Masked lanes/tiles/threads MUST NOT raise exceptions or produce side-effects.
3. Memory-visible operations (DMA, loads/stores, gather/scatter, atomics, streaming, scratchpad moves) MUST NOT issue memory accesses for masked lanes/tiles/threads, consistent with `XPHMG_XMEM`.
4. Writeback for predicate-false elements is governed exclusively by `CAP.PREC.MODE.ZMODE`:
   - `merge`: masked elements preserve the previous destination value.
   - `zero`: masked elements write zero.

"Destination" in ACCEL contexts means the architecturally visible per-element result storage of the operation (e.g., register file, accumulator/tile register, or result buffer). ACCEL MUST NOT define its own predicate source or writeback policy beyond PMASK and `CAP.PREC.MODE.ZMODE`.

### 7.5 Memory rules

1. ACCEL loads/stores must obey `CAP.XMEM.DOM`, `CAP.XMEM.L1CFG`, `CAP.XMEM.CLSMAP`, `CAP.XMEM.LDS_BUDGET`, streaming profiles, and XMEM event semantics.
2. Indexed or indirect addressing used by ACCEL must use the same SVMEM configuration (`CAP.XMEM.SVM*`) as RSV/MTX/GFX/RT. Predicated memory behavior follows Section 7.4 and `XPHMG_XMEM`.
3. ACCEL-specific memory failures (e.g., out-of-bounds, misalignment, LDS out-of-memory) must set appropriate bits in `CAP.XMEM.EVENTS` for unified telemetry.

### 7.6 Interaction with descriptor queues

- Descriptor linkage via `next` is singly-linked. Cycles or malformed chains are undefined and may result in vendor-defined error signaling.
- Doorbell writes that reference descriptors must ensure descriptors are fully written and visible per platform memory ordering before notification. ACCEL assumes descriptors are immutable after doorbell unless the platform ABI defines otherwise.

NOTE: If a platform defines descriptor-level fences or flags that alter ordering, those semantics must be documented alongside the flag definitions. This specification does not introduce new fence primitives beyond CAP/XMEM behavior.

---

## 8. Exceptions, Errors, and Status (Normative)

This section defines how ACCEL reports completion, errors, and exceptions in coordination with CAP and XMEM.

### 8.1 Descriptor status

Each accelerator queue must have a simple status word per descriptor, for example:

```c
struct XPHMG_ACCEL_STATUS {
  uint32_t done   : 1;  // 1 = completed
  uint32_t error  : 1;  // 1 = error, see code
  uint32_t code   : 6;  // error code (OOB, UNSUP_FMT, TIMEOUT, etc.)
  uint32_t rsv0   : 24;
  uint32_t cycles;      // optional: approximate cost
};
```

The ABI for exposing this status (MMIO vs memory) is platform-defined, but must satisfy:

- Errors caused by numeric policy must correlate with `CAP.PREC.EXC.ST` bits (e.g., NaNs, overflow/underflow).
- Errors caused by memory policy must correlate with `CAP.XMEM.EVENTS` bits.
- The `code` field is vendor-defined. Mappings to numeric or memory policy violations must be documented and consistent with CAP/XMEM event bits when applicable.

### 8.2 Error signaling and routing

- ACCEL errors and exceptions must be reported through the same architectural surfaces used by other XPHMG domains:
  - Numeric exceptions: `CAP.PREC.EXC.ST`, honoring `CAP.PREC.EXC.EN` and SAE policy.
  - Memory errors/events: `CAP.XMEM.EVENTS`.
  - Optional queue/descriptor status: per-descriptor status word and/or status CSR.
- Interrupt delivery for ACCEL completion/error is platform-defined. If supported, enable/mask/ack controls must follow Section 5 requirements.

### 8.3 Unsupported or malformed descriptors

- If a descriptor requests unsupported precision (`prec_mode`), memory policy, or opcode encoding, the implementation must either:
  - Reject/trap the descriptor as Illegal Instruction / Unsupported Descriptor (vendor choice); or
  - Execute with the nearest supported mode and surface `UNSUP_FMT` or an equivalent error code in the status word and relevant CAP/XMEM bits.
- Descriptors with malformed linkage (cycles or invalid `next`) are undefined. Vendors may signal an error code and halt queue consumption.

### 8.4 Completion ordering

- Completion signaling (status word update, CSR update, interrupt) must occur only after all architecturally visible side-effects of the descriptor are complete under the selected XMEM domain/class semantics.

NOTE: Exact error code assignments and interrupt vectors are vendor-defined. They must maintain the correlation requirements above and must not introduce new architectural exception classes beyond CAP/XMEM.

---

## 9. Debug and Deterministic Replay (Normative)

This section defines the minimum observability and replay guarantees for ACCEL to support debugging and deterministic reproduction of workloads.

### 9.1 Deterministic replay

To support deterministic replay:

1. Debuggers must be able to dump `CAP.PREC.STAT`, `CAP.XMEM.*`, and the descriptor contents at job boundaries.
2. Engines must not expose partially updated CAP/XMEM or descriptor state during a job.
3. Vendors may implement shadow mirror registers for internal scheduling. These are micro-architectural and must remain coherent with CAP at descriptor boundaries.

### 9.2 Trace and observability requirements

- Implementations must provide a way to observe descriptor boundaries (e.g., via status CSRs or completion records) sufficient to correlate hardware execution with software queues.
- If performance counters or debug CSRs are provided (see Section 5.3), they must reflect architecturally visible behavior and must not expose speculative state beyond what is observable via CAP/XMEM events.
- Vendor-specific debug visibility (e.g., mirror registers) must not alter architectural ordering or state.

### 9.3 Ordering and timestamping

- Ordering of descriptors follows Section 7. Debug and replay mechanisms must not reorder or elide descriptor boundaries.
- Timestamping, if provided, must be monotonic per ACCEL domain and sufficient to correlate with descriptor completion order.

NOTE: Exact debug CSR definitions and trace formats are vendor-defined. They must respect the architectural visibility and ordering guarantees described above.

---

## 10. Capability Discovery and Versioning (Normative)

This section defines how ACCEL capabilities are discovered via CAP surfaces and how versioning and compatibility are expressed.

### 10.1 Suggested Capability Bits (in CAP.FEAT / CAP.TIER)

While `XPHMG_ACCEL` itself does not define new CSRs, it recommends exposing accelerator-related capabilities via `CAP.FEAT*` and `CAP.TIER.*`, for example:

- `CAP.FEAT1.ACCEL_PRESENT` - generic "ACCEL/NPU present".
- `CAP.FEAT1.ACCEL_DESC_CHAIN` - supports descriptor chaining via `next`.
- `CAP.FEAT1.ACCEL_MEMCTL` - supports embedded `XPHMG_MEMCTL` block.
- `CAP.TIER.ACCEL` (optional CSR) - capability tier (e.g., MAC/s, maximum workgroup size).

Exact bit assignments are vendor-specific but should be documented aligned with the CAP specification.

### 10.2 Compatibility

- ACCEL uses `XPHMG_CAP` as the sole source of numeric policy.
- ACCEL uses `XPHMG_XMEM` as the sole source of architectural memory policy.
- Any vendor-specific ACCEL knobs should map to descriptor fields (e.g., `prec_mode`, `xmem_class`, `sprof_id`) or remain purely micro-architectural.

### 10.3 Versioning and profiles

- The ACCEL extension version is indicated in the preamble. Backward-compatible revisions must preserve descriptor layout and semantics or provide explicit migration guidance.
- Vendors should document deviations or extensions (e.g., additional descriptor formats) and tie them to CAP.FEAT* / CAP.TIER.* so software can feature-detect and select compatible paths.
- ACCEL is intended to coexist with other XPHMG extensions. No additional profiles are defined beyond the capabilities advertised via CAP.FEAT* / CAP.TIER.*.

NOTE: Exact bit positions and tier encodings are vendor-defined but must be stable for a given version and documented alongside the ACCEL implementation.

---

## 11. Security and Isolation Considerations (Informative)

This section highlights security and isolation considerations for ACCEL implementations that build on CAP/XMEM.

- DMA containment and protection: ACCEL engines fetch descriptors and operate on memory via the same translation and protection mechanisms as other XPHMG domains. Platform PMP/IOPMP (or equivalent) must be applied to ACCEL DMA and queue/completion buffers to prevent unauthorized access.
- Coherence/domain isolation: Selection of `dom` and XMEM classes determines visibility of ACCEL memory operations. Software should assign domains/classes to avoid unintended sharing across tenants or privilege domains.
- Virtualization: In virtualized environments, descriptor queues and doorbells must be mediated so guests cannot target host or peer queues. CAP/XMEM state must be virtualized per guest to avoid cross-tenant leakage of numeric or memory policy.
- Side-channel considerations: ACCEL implementations should consider timing and resource contention channels (e.g., shared caches, LDS budgets). This specification does not define side-channel mitigations; platforms may need to provide partitioning or throttling mechanisms.
- Descriptor integrity: Descriptors are trusted inputs to ACCEL. Malformed or adversarial descriptors must not allow escalation beyond the behavior described in Section 8 (e.g., must not corrupt CAP/XMEM state or other tenants' queues).

NOTE: Specific mitigation mechanisms (e.g., cache partitioning, constant-time execution) are platform choices and outside the scope of this v0.1 specification.

---

## Appendix A. Programming Examples (Informative)

NOTE: Content moved from the Programmer's Model section for clarity. These examples illustrate descriptor construction and submission patterns and do not add normative requirements.

A typical NPU/ACCEL workflow:

1. Program CAP/XMEM once per phase

   ```asm
   # Example: FP16 compute, FP32 accumulate, INT8 quantized path enabled
   # (see XPHMG_CAP for exact encoding)
   csrw 0x7D0, t0       # CAP.PREC.MODE
   csrw 0x7D1, t1       # CAP.PREC.ALT (optional FP8/INT4 modes)
   csrw 0x7E2, t2       # CAP.XMEM.L1CFG
   csrw 0x7E3, t3       # CAP.XMEM.CLSMAP
   csrw 0x7E7, t4       # CAP.XMEM.STREAM_CTL
   ```

2. Build descriptors in memory

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

3. Submit to queue

   - Write descriptor pointer to a queue tail register.
   - Ring a doorbell CSR (implementation-defined).
   - Hardware consumes descriptors until `next==0` or queue empty.

4. Wait for completion

   - Poll a status CSR; or
   - Use interrupts based on `flags.INT_EN`; or
   - Use memory-mapped completion records.

NOTE: Platforms may provide additional steps (e.g., programming MEMCTL blocks or enabling ACCEL-specific interrupts) consistent with the normative sections. Such steps must not contradict the CAP/XMEM policy model.

---
**License:** CC-BY-SA 4.0 - Open Specification maintained by Pedro H. M. Garcia.
