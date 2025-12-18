# RISC-V Vendor Extension Family **XPHMG**
**Heterogeneous Graphics / Matrix / Ray / Vectors / NPU**

**Version:** 0.1.0  
**Author:** Pedro H. M. Garcia  
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview

**XPHMG** is a family of RISC-V vendor extensions that defines a unified **execution substrate** for modern graphics, geometry, and compute workloads.

Rather than exposing a fixed graphics pipeline or a driver-managed API, XPHMG provides a small set of **orthogonal architectural building blocks** that software composes explicitly:

* **CAP** - authoritative capability discovery and architectural numeric/memory policy.
* **XMEM** - explicit control over LDS, cache behavior, streaming, and cross-domain data residency.
* **RSV** - scalar-vector reinterpretation for compact and predictable vectorization.
* **MTX** - small matrix and tile compute primitives.
* **RT** - ray and geometry intersection primitives.
* **GFX** - texture, interpolation, shading, and visibility helpers.
* **ACCEL** - descriptor-driven offload for specialized accelerators.

XPHMG is **not** a graphics API, a driver model, or a fixed-function pipeline.
It does not perform implicit scheduling, state inference, or automatic memory promotion.
All observable behavior - numeric precision, masking, memory residency, coherence domain, and execution ordering - is **explicitly controlled by architectural state**.

### 1.1 Conceptual Model (Informative)

* XPHMG defines **multiple execution domains** that share a single architectural substrate for precision and memory (`XPHMG_CAP` + `XPHMG_XMEM`).
* Each domain consumes the same architecturally visible state at instruction boundaries; domain-local mirrors are micro-architectural and MUST NOT diverge in architecturally visible behavior.
* Cross-domain dataflow is explicit. Ownership, residency, and streaming behavior are described via XMEM classes/domains and, when applicable, `XPHMG_MEMCTL` descriptor tails.
* No implicit pipeline modes exist; pipelines (e.g., MTX -> RT -> GFX) are expressed by explicit instruction streams and descriptors.

### 1.1.1 Architectural Guarantees & Invariants (Informative)

* **Single source of truth:** Numeric policy (`CAP.PREC.*`) and memory policy (`CAP.XMEM.*`) are authoritative for all domains; no domain may reinterpret them.
* **Instruction-boundary latching:** Domains snapshot effective CAP/XMEM state at decode; mid-instruction writes MUST NOT affect in-flight operations (see `XPHMG_CAP`).
* **Determinism:** Given identical instruction streams and identical CAP/XMEM state, execution is deterministic; stickies and status reflect the effective state used.
* **Non-binding hints:** Capability and tier/hint CSRs are descriptive and non-binding; correctness MUST NOT depend on them.

### 1.1.2 Interaction with CAP / XMEM / Other Extensions (Informative)

* `XPHMG_CAP` exposes precision and memory policy consumed by all domains.
* `XPHMG_XMEM` provides the architectural memory model (classes, domains, streaming, gather/scatter); all domains use the CAP-sourced XMEM state at decode.
* `XPHMG_RSV` mask semantics and SVMEM gather/scatter follow `CAP.PREC.MODE.ZMODE` and `CAP.XMEM.SVM*`.
* `XPHMG_MTX`, `XPHMG_RT`, and `XPHMG_GFX` inherit precision/memory policy; optional descriptor tails embed `XPHMG_MEMCTL` for domain/class/stream hints.
* `XPHMG_NPU` descriptors reuse CAP precision policy and the XMEM memory model; submission is via CSR doorbells defined in the NPU spec.

### 1.1.3 Undefined / Reserved Behavior (Informative)

* Behavior is UNDEFINED if software relies on mid-instruction visibility of CAP/XMEM updates or assumes advisory fields change architectural correctness.
* Behavior is UNDEFINED if domains execute with diverged mirrors of CAP/XMEM state.
* NOTE: This behavior is intentionally left unspecified in v0.1 and may be clarified in a future revision - handling of partial descriptor programming when submission is aborted mid-sequence.

### 1.1.4 Notes for Implementers (Informative)

* Program `CAP.PREC.*` and `CAP.XMEM.*` per phase, not per instruction; stage writes then apply (PREC) to avoid transient states.
* Engines may implement micro-architectural mirrors of CAP/XMEM for timing, but must refresh them at each instruction boundary.
* When composing pipelines, use `XPHMG_MEMCTL` tails to align class/domain/stream policy across RT/NPU/GFX/MTX submissions.

### 1.2 Design Principles

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

### 1.3 Execution Domains

XPHMG extensions operate as **execution domains** that share a common architectural substrate:

* Domains may execute independently or cooperatively.
* Cross-domain dataflow is performed explicitly through XMEM-described regions.
* Logical ownership and priority are expressed via memory class and domain metadata, not by physical copies.

This model enables efficient pipelines such as MTX -> RT -> GFX or compute-only workflows without requiring implicit synchronization or opaque command processors.

### 1.4 Memory and Tiling Model

XPHMG naturally supports **tile-based rendering and tiled compute** through:

* Explicit LDS/XMEM allocation,
* Tiered memory residency (Tier-0 / Tier-1 / higher tiers),
* Domain-controlled ownership and handoff,
* Canonical tile layouts (XDL) shared across engines.

Tiles are a **software-defined working set**, not an architectural object.
Their size, lifetime, and layout are fully controlled by software.

### 1.5 Scope of Version 0.1

Version **0.1** defines the **first stable architectural baseline** of XPHMG.

It establishes:

* The CAP authority model,
* The unified XMEM memory semantics,
* The execution-domain framework,
* The instruction-level contracts of RSV, MTX, RT, GFX, and ACCEL.

Future revisions may extend capabilities or add optional features, but **must preserve the architectural guarantees defined in this baseline unless explicitly stated**.

---

## 2. Extension Map and Dependencies

This section is informative. It enumerates the XPHMG family members, their roles, and their dependency ordering. All domains consume `XPHMG_CAP` (precision and policy) and `XPHMG_XMEM` (memory model) at instruction boundaries; domain-local mirrors are micro-architectural only.

| Extension            | Category          | Scope                                      | Depends on                         |
|----------------------|-------------------|--------------------------------------------|------------------------------------|
| `XPHMG_CAP`          | Vendor Extension  | Capabilities, numeric & memory policy      | -                                  |
| `XPHMG_XMEM`         | Vendor Extension  | LDS/cache/streaming model + mem ISA        | `XPHMG_CAP`                        |
| `XPHMG_RSV`          | Vendor Extension  | Scalar-Vectors scalar-vector layer         | `XPHMG_CAP`, `XPHMG_XMEM`          |
| `XPHMG_RSV-Profiles` | Vendor Extension  | RSV permute/compact/saturating profiles    | `XPHMG_RSV`, `XPHMG_CAP`           |
| `XPHMG_MTX`          | Vendor Extension  | Small MxN matrix/tensor core               | `XPHMG_CAP`, `XPHMG_XMEM`          |
| `XPHMG_RT`           | Vendor Extension  | Ray/AABB/Triangle, BVH, optional RT offload| `XPHMG_CAP`, `XPHMG_XMEM`          |
| `XPHMG_GFX`          | Vendor Extension  | Graphics/texture pipeline ops              | `XPHMG_CAP`, `XPHMG_XMEM`, `XPHMG_RSV` (recommended) |
| `XPHMG_NPU`          | Descriptor Format | GEMM/CONV/POOL/ELTWISE descriptors         | `XPHMG_CAP`, `XPHMG_XMEM`          |

**Informative constraints:**

1. `XPHMG_CAP` and `XPHMG_XMEM` form the architectural base. All other domains rely on their state; no domain redefines CAP/XMEM semantics.
2. `XPHMG_RSV` is the recommended vector substrate for `XPHMG_GFX` and can be composed with MTX/RT/GFX pipelines without redefining precision/memory rules.
3. Descriptor-driven engines (`XPHMG_RT`, `XPHMG_NPU`, optional GFX/MTX offload) carry `XPHMG_MEMCTL` tails to align class/domain/stream policy with CAP/XMEM.

**Programming model rule:**  
At decode time, any XPHMG engine must observe:

1. The **effective precision state** from `CAP.PREC.MODE/ALT/STAT`.
2. The **effective memory policy** from `CAP.XMEM.*` (domains, classes, streaming, SVMEM).

Engine-local mirrors are allowed but non-architectural.

### 2.1 Mandatory vs Optional Components

* **Mandatory base for any XPHMG implementation:** `XPHMG_CAP` and `XPHMG_XMEM`.
* **Recommended for vector/graphics pipelines:** `XPHMG_RSV`; `XPHMG_RSV-Profiles` are optional extensions to RSV behavior.
* **Domain add-ons:** `XPHMG_MTX`, `XPHMG_RT`, `XPHMG_GFX`, `XPHMG_NPU` may be implemented independently, provided they consume CAP/XMEM state as specified in their respective documents.
* TODO-REFINE: Confirm profile-specific minimums for each published implementation profile (see section 6).

---

## 3. Architectural Substrate Summaries

This section is informative. It summarizes each XPHMG substrate and its architectural role. Normative bit-level definitions reside in the respective extension specifications.

### 3.1 Capabilities & Precision (`XPHMG_CAP`)

**Purpose:** single source of truth for capability discovery, numeric policy, and architectural memory defaults.

**Scope:** `CAP.FEAT*`, `CAP.TIER.*`, and `CAP.HINT.*` are advisory (RO); `CAP.PREC.*` and `CAP.XMEM.*` are architectural state (RW where defined).

**Invariants (informative):**
- Domains MUST consume `CAP.PREC.*` and `CAP.XMEM.*` latched at instruction boundary; mid-instruction updates are not observed.
- Advisory CSRs MUST NOT be used as correctness preconditions; undefined bits read zero and ignore writes.

**Interactions:**
- `CAP.PREC.MODE/ALT/STAT` define effective PET/EW/ACCW, rounding, FTZ, NaN policy, ZMODE; all domains inherit these defaults.
- `CAP.XMEM.*` provides default classes/domains/streaming/gather-scatter; descriptor tails (`XPHMG_MEMCTL`) override per job where defined.

> For normative field definitions, see `XPHMG_CAP`.

### 3.2 Unified Memory / LDS / Streaming (`XPHMG_XMEM`)

**Purpose:** defines the architectural memory model (classes, domains, LDS/cache/streaming) and memory instructions (`XMEM.*`, `XLDS.*`, `XSWZ.*`).

**Invariants (informative):**
- `CAP.XMEM.*` is the authoritative default state; XMEM mirrors (if implemented) must match at instruction boundaries.
- LDS/class/domain semantics, streaming profiles, and SVMEM configuration MUST match CAP defaults unless explicitly overridden by descriptors or per-instruction controls defined in the XMEM spec.

**Interactions:**
- `XPHMG_MEMCTL` tails in descriptors (RT/NPU/GFX/MTX offload) override CAP defaults for domain/class/stream/security when present.
- RSV gather/scatter, MTX/GFX/RT loads/stores, and NPU descriptors all adhere to XMEM ordering/fence rules.

> Instruction encodings and CSR layouts are defined in `XPHMG_XMEM`.

### 3.3 Scalar-Vectors (`XPHMG_RSV` + profiles)

**Purpose:** provides a scalar-vector reinterpretation layer over scalar ALU/LSU, sharing CAP precision and XMEM memory policy.

**Invariants (informative):**
- Masks follow `CAP.PREC.MODE.ZMODE` (merge vs zeroing).
- Gather/scatter uses `CAP.XMEM.SVM*` defaults; FOF/fault index semantics are defined in RSV.
- FP rounding/SAE/ZMODE may be overridden per instruction via RSV controls; defaults revert after the instruction.

**Optional profiles:** `XPHMG_RSV-Profiles` add permute/compact/saturating behaviors and reuse RSV mask/precision/memory semantics. Unsupported profiles are optional; bits are discoverable via RSV capability fields.

> See `XPHMG_RSV` and `XPHMG_RSV-Profiles` for encodings and CSR details.

### 3.4 Matrix/Tensor Core (`XPHMG_MTX`)

**Purpose:** small-tile matrix/tensor execution using CAP precision and XMEM data movement.

**Invariants (informative):**
- Precision behavior (PET/EW/ACCW, SAT, PACK, Q) is derived from `CAP.PREC.*`; MTX must not redefine it.
- Data movement uses XMEM/LDS; no bespoke DMA semantics outside XMEM.

**Notes:** Capability discovery (tile shapes, bias support, wide/native tiles) is via `MTXCAP`; unsupported shapes must be handled per MTX spec (e.g., software tiling).

> Instruction encodings and CSR layouts are defined in `XPHMG_MTX`.

### 3.5 Ray/Geometry (`XPHMG_RT`)

**Purpose:** scalar ray-box/triangle primitives, BVH traversal, and optional descriptor-driven offload.

**Invariants (informative):**
- Numeric behavior (precision, rounding, FTZ, pack hints) is inherited from CAP; no RT-specific reinterpretation.
- Memory behavior (LDS allocation, class, streaming profile) follows XMEM and `XPHMG_MEMCTL` in descriptors.
- BVH node layouts are fixed and shared across domains that consume RT-produced data.

**Notes:** Offload descriptors (`XPHMG_RT_DESC`) embed `XPHMG_MEMCTL` to align memory/class/stream policy with CAP/XMEM defaults.

> Instruction encodings, descriptor formats, and BVH layout details are defined in `XPHMG_RT`.

### 3.6 Graphics / Texture (`XPHMG_GFX`)

**Purpose:** instruction-level graphics/texture pipeline sharing the CAP/XMEM substrate.

**Invariants (informative):**
- Precision and memory policy are inherited from CAP/XMEM; GFX must not introduce alternate numeric/memory rules.
- Vectorization typically relies on RSV; gather/scatter and masking follow CAP/XMEM and RSV semantics.

**Notes:** Implementations may map sampling/interpolation to LDS-first, class-aware paths; layout shuffles/visibility compaction may use RSV profiles.

> Instruction set and CSR details are defined in `XPHMG_GFX`.

### 3.7 NPU / Tensor Offload (`XPHMG_NPU` descriptor)

**Purpose:** descriptor-only path for GEMM/CONV/POOL/ELTWISE and related kernels.

**Invariants (informative):**
- Precision policy (PET/EW/ACCW, SAT, Q) is inherited from CAP; descriptors do not redefine numeric semantics.
- Memory model (LMEM/DRAM, class/domain/stream) follows XMEM; descriptors carry a `XPHMG_MEMCTL` tail to override CAP defaults per job.

**Notes:** Doorbell CSRs (`NPUCFG`, `NPUDESC`, `NPUSTAT`, `NPUEVENT`, `NPUMAX`) provide submission/completion; op-specific bodies are defined in the NPU specification.

> Descriptor layout and semantics are defined in `XPHMG_NPU`.

---

## 4. Interoperability and Cross-Domain Guarantees (Normative)

This section defines cross-domain guarantees required for consistent behavior across all XPHMG execution domains (RSV, MTX, RT, GFX, NPU). It makes explicit how CAP and XMEM state are consumed and how domains interoperate without redefining numeric or memory semantics.

### 4.1 Precision Interoperability (CAP -> All Domains)

* Engines MUST snapshot the effective numeric state at the instruction boundary, as defined by `CAP.PREC.MODE`, `CAP.PREC.ALT`, and `CAP.PREC.STAT`. Mid-instruction writes MUST NOT affect in-flight operations.
* Effective PET/EW/ACCW, saturation, rounding, FTZ, NaN policy, ZMODE, SAE, quantization, and packing apply uniformly to RSV, MTX, RT, GFX, and XMEM gather/scatter.
* Per-instruction overrides (e.g., RSV `svon.fpctl`) take effect after CAP defaults and revert for the next instruction.
* Stickies (`SAT_HIT`, `FTZ_HIT`, `UNSUP_FMT`, `DOWNCAST_TAKEN`) and exception stickies follow CAP rules and MUST be updated consistently across domains.

### 4.2 Unified Memory Semantics (CAP.XMEM -> All Domains)

* `CAP.XMEM.*` is the architectural source of truth for LDS allocation/budgets, class partitioning, domain selection, cache behavior, streaming profiles, gather/scatter parameters, and memory ordering (see `XPHMG_XMEM`).
* Engine-local mirrors (if any) MUST be coherent with CAP at every instruction boundary; divergence in architecturally visible behavior is not permitted.
* Descriptor tails (`XPHMG_MEMCTL`) may override CAP defaults per job (domain/class/stream/security) where defined in the consuming extension; absent overrides, CAP defaults apply.
* Gather/scatter (SVMEM) semantics?including mask, scale, sign, FOF, and fault index reporting?MUST match CAP/XMEM definitions for RSV, MTX, RT, GFX, and NPU paths.

### 4.3 Cross-Domain Dataflow & Tile Handoff (Informative)

XPHMG domains target complementary scales; none supersedes another:

* **RSV**: lane-local vectorization, permute/compact, gather/scatter helpers (`XPHMG_RSV`).
* **MTX**: small-tile matrix MACs and mixed precision (`XPHMG_MTX`).
* **RT**: scalar ray-box/triangle tests and BVH traversal (`XPHMG_RT`).
* **GFX**: shading, sampling, and raster helpers (`XPHMG_GFX`, optional).

Shared CAP/XMEM state enables:

* MTX to consume RSV-produced vectors without reformatting.
* RT to use MTX outputs (e.g., transforms) without reprogramming precision.
* GFX to use RSV permutes for layout changes without extra copies.
* Consistent masked-lane semantics via `CAP.PREC.MODE.ZMODE` across domains.

#### 4.3.1 Unified Tile Handoff Model (Informative)

* Tiles produced into a Tier-0 shared class (e.g., `CLS_SHARED`) use canonical XDL layout.
* Ownership of the tile is transitioned explicitly (e.g., `DOM_RSV`, `DOM_GFX`, `DOM_RT`) via XMEM class/domain controls or descriptor metadata.
* Consumers read the tile directly from XMEM; no architectural copies or reinterpretation are implied.

### 4.4 Undefined / Reserved Behavior (Informative)

* Behavior is UNDEFINED if software relies on mid-instruction visibility of CAP/XMEM updates or on diverged engine-local mirrors.
* Behavior is UNDEFINED if software assumes advisory/hint fields alter correctness.
* NOTE: This behavior is intentionally left unspecified in v0.1 and may be clarified in a future revision ? handling of partially programmed descriptors that are not subsequently submitted.

### 4.5 Notes for Implementers (Informative)

* Program CAP/XMEM per phase; stage writes and rely on boundary latching (PREC via APPLY, XMEM at decode) to avoid transient states.
* Use descriptor tails (`XPHMG_MEMCTL`) to align domain/class/stream policy across RT/NPU/GFX/MTX submissions; absent tails, CAP defaults apply.
* Ensure debug/replay visibility reflects the effective CAP/XMEM state used for execution (including latched exception enables).

---

## 5. Programming / Submission Model (Informative/Recommended)

This section is informative. It describes the recommended submission structure for XPHMG domains so that software stacks remain coherent. Normative instruction and CSR definitions reside in the respective domain specifications.

### 5.1 Common Descriptor Structure (Informative)

* **Header (common):** size, version, opcode/type, flags, precision override (if permitted by the domain), completion pointer, user tag.
* **Body (domain-specific):** fields defined by the target domain (e.g., RT BVH pointers/opts, NPU op bodies, GFX/MTX job parameters).
* **Tail (`XPHMG_MEMCTL`):** optional but recommended; carries domain/class/stream/security hints to align with `CAP.XMEM.*` defaults. Absent a tail, CAP defaults apply.

NOTE: This behavior is intentionally left unspecified in v0.1 and may be clarified in a future revision ? whether all domains must accept a uniform descriptor header size vs. allowing domain-specific header variants.

### 5.2 Domain Submission Paths (Informative)

* **RT:** `XPHMG_RT_DESC` plus `XPHMG_MEMCTL` tail; submitted via RT doorbells as defined in `XPHMG_RT`.
* **NPU:** 64-byte header + op-specific body + `XPHMG_MEMCTL` tail; submitted via `NPUCFG/NPUDESC` doorbells.
* **GFX/MTX (optional offload):** may adopt the same header/body/tail pattern to align with RT/NPU for a unified software stack. TODO: Clarify interaction with `XPHMG_GFX` submission if a descriptor model is adopted.
* **RSV/inline use:** RSV and other inline instruction streams consume CAP/XMEM directly; descriptors are not required.

### 5.3 Architectural Guarantees & Invariants (Informative)

* All domains that accept descriptors MUST consume CAP/XMEM state at the instruction boundary; descriptors do not override CAP precision unless the domain spec explicitly allows a per-job precision field.
* `XPHMG_MEMCTL` tails override only the fields defined in XMEM (domain/class/stream/security). Undefined tail bits are ignored; absent tails revert to CAP defaults.
* Submission ordering, fences, and visibility follow the XMEM memory model; doorbell writes are ordinary stores unless the domain spec defines additional ordering.

### 5.4 Undefined / Reserved Behavior (Informative)

* Behavior is UNDEFINED if software assumes partially written descriptors will be interpreted; descriptors must be fully programmed before submission.
* Behavior is UNDEFINED if software assumes descriptor fields can override CAP precision semantics unless the domain spec permits it.
* NOTE: This behavior is intentionally left unspecified in v0.1 and may be clarified in a future revision ? whether descriptor submission is allowed while CAP/XMEM writes are in-flight.

### 5.5 Notes for Implementers (Informative)

* Align descriptor layouts across domains (RT, NPU, optional GFX/MTX) to simplify toolchains and firmware.
* Keep CAP/XMEM programming per phase; avoid per-job reprogramming unless required by workload changes.
* Ensure debug/replay capture includes both the submitted descriptor and the effective CAP/XMEM state at submission/decode.

---

## 6. Implementation Profiles

This section is informative. It defines reference implementation groupings to aid software/toolchain configuration and silicon scoping. Profiles do not change architectural semantics; they bundle extensions and expected capability baselines.

### 6.1 Profile Definitions (Informative)

* **XPHMG-Base:** `XPHMG_CAP + XPHMG_XMEM`. Minimum substrate required for any other XPHMG domain.
* **XPHMG-RV-Vector:** Base + `XPHMG_RSV` (+ minimal RSV profiles as implemented). Suitable for scalar-vector pipelines without graphics/RT.
* **XPHMG-GFX:** Base + XMEM + RSV + GFX. Assumes RSV is present for vectorization; optional RSV profiles may enhance layout/compaction.
* **XPHMG-RTU:** Base + XMEM + RT (+ MTX recommended for transforms/denoisers). RSV is optional but recommended for data prep/compaction.
* **XPHMG-NPU:** Base + XMEM + NPU descriptors (no GFX/RT required). RSV/MTX optional depending on software pipeline.
* **XPHMG-Hybrid:** Base + XMEM + RSV + MTX + RT (+ optional GFX/NPU). Intended for heterogeneous workloads combining graphics, RT, and ML.

### 6.2 Capability/Format Expectations (Informative)

For each profile, software should document at least:

* Supported `CAP.PREC.*` formats (e.g., FP16/FP32 mandatory? FP8/INT4 optional?).
* XMEM capabilities (LDS presence/size, class partitioning, compression support, streaming profiles).
* RSV profiles implemented (permute/compact/saturating) and domains present (GFX/RT/MTX/NPU).
* Any optional domain-specific features (e.g., RT offload descriptors, MTX wide/native tiles, NPU post-ops).

TODO: Clarify per-profile minimums (e.g., which CAP precision formats are required for ?-GFX? vs ?-NPU?) in a future revision.

### 6.3 Notes for Implementers (Informative)

* Profiles are descriptive only; correctness derives from CAP/XMEM and domain specifications. Absence of a profile does not prohibit implementing a superset.
* Toolchains may use profiles to select default codegen/tiling/precision strategies; runtimes must still query CAP/FEAT/TIER/HINT for precise tuning.
* Silicon vendors should publish which optional features are absent to avoid silent downgrade assumptions (e.g., no FP8, no compression, limited LDS).

---

## 7. Versioning and Change Management

This section is informative. It describes how versioning is reported and how forward-compatible behavior is expected to work across XPHMG extensions.

### 7.1 CAP/XMEM Version Reporting (Informative)

* `XPHMG_CAP` exports `CAP.VERS` for the CAP/XMEM layout. Software MUST treat unknown fields as read-zero/ignored-writes, per the CAP spec.
* Undefined/reserved bits in CAP/XMEM CSRs MUST return zero and ignore writes; extensions MUST NOT redefine CAP/XMEM semantics.

### 7.2 Per-domain Versioning (Informative)

* Each domain spec (`RSV`, `RSV-Profiles`, `XMEM`, `MTX`, `RT`, `GFX`, `NPU`) publishes its own version and change log.
* Domain-specific capability CSRs (e.g., `MTXCAP`, `CAP.PREC.*`, RSV capability bits) are the authoritative source for feature discovery; unknown bits read zero and are ignored by software.

### 7.3 Forward Compatibility Expectations (Informative)

* Precision formats or XMEM capabilities not recognized by software MUST be treated as unsupported/ignored; hardware sets `UNSUP_FMT` in `CAP.PREC.STAT` for precision downgrades as defined in `XPHMG_CAP`.
* Toolchains and runtimes MUST NOT assume availability of optional formats/features unless indicated by capability bits; absence implies "not supported or not exposed."
* NOTE: This behavior is intentionally left unspecified in v0.1 and may be clarified in a future revision ? how per-domain minor version mismatches (e.g., MTX vs CAP) should be surfaced to software beyond capability bits.

### 7.4 Notes for Implementers (Informative)

* Publish version numbers and change logs per domain alongside CAP/XMEM version to aid driver/toolchain gating.
* Ensure unknown/unsupported bits follow read-zero/ignore-writes behavior to avoid regressions across revisions.

---

## Appendix A. Guidelines and Tips (Informative)

This appendix is informative. It consolidates practical guidance for software, hardware, and runtime authors to maintain alignment with the architectural rules defined elsewhere in this specification.

### A.1 Software Guidelines

1. **Program CAP per phase.** Group work by precision/mode; avoid per-instruction writes to `CAP.PREC.*` and `CAP.XMEM.*`. Use APPLY (PREC) and instruction-boundary latching (XMEM) to avoid transient states.
2. **Use XMEM classes intentionally.** Partition CL0/CL1/CL2/CL3 to match pipeline roles (e.g., RT/MTX/HYB vs. GFX/compute) and respect budgets/hints.
3. **Use RSV for lane-local work.** Prefer RSV for small vectors/structs; use RSV profiles (if present) for permutes/compaction/saturating arithmetic.
4. **Use descriptors when possible.** Let RT/NPU (and optional GFX/MTX offload) overlap compute and data movement; include `XPHMG_MEMCTL` tails to align with CAP/XMEM.
5. **Check capabilities, do not assume.** Query `CAP.FEAT*`, `CAP.TIER.*`, and domain capability CSRs before using optional formats/features.

### A.2 Hardware Guidelines

1. **Latch CAP at instruction boundaries.** Mid-instruction CAP writes must not affect in-flight operations.
2. **Treat CAP.XMEM as architectural.** Mirrors are micro-architectural; refresh at each instruction boundary.
3. **Implement FOF/fault reporting consistently.** SVMEM fault-only-first and `SVFAULTI` behavior must match RSV definition.
4. **Favor robust accumulation.** Use FP32 accumulators internally for FP8/FP16/BF16 unless area/power constraints dominate.
5. **Honor ZMODE uniformly.** Masked-lane writeback must follow `CAP.PREC.MODE.ZMODE` across domains that implement masking.

### A.3 Common Pitfalls

1. **Non-latched CAP state:** Latch CAP precision/memory at decode; never allow mid-instruction visibility.
2. **Inconsistent gather/scatter semantics:** Respect SVMEM masks, `FOF`, zero/merge masked lanes, and index scale/sign across domains.
3. **LDS bank conflicts / soft budgets:** Mispartitioned classes can serialize RT/MTX; follow `CAP.XMEM.LDS_*` and class budgets.
4. **Poor integration of RSV permutes with MTX:** Use RSV permutes to place tiles in LDS in the layout expected by MTX before execution.
5. **Ignored stickies:** MTX, RSV, scalar ALU, and RT must propagate stickies to `CAP.PREC.EXC.ST` / `CAP.PREC.STAT`.
6. **Assuming hints are binding:** Tiers/hints are descriptive; correctness must not depend on them.

### A.4 Implementation Checklists

#### A.4.1 For hardware designers

* Enforce boundary-based CAP latching for PREC/XMEM.
* Provide a single shared “effective precision” structure consumed by all domains.
* Keep XMEM mirrors (if any) coherent at every instruction boundary.
* Validate consistent SVMEM addressing/masking across RSV, MTX, RT, and GFX.

#### A.4.2 For driver/runtime authors

* Program CAP once per phase; avoid oscillating precision/memory state between adjacent jobs.
* Configure XMEM classes to match workload phases; use fences per XMEM rules when transitioning.
* Use RSV permutes to prepare data layouts for MTX/GFX; avoid redundant copies.
* Use descriptor tails (`XPHMG_MEMCTL`) to steer domain/class/stream/security for NPU/RT/GFX/MTX submissions.
* Capture CAP/XMEM state alongside descriptors for replay/debug.

---

## Appendix B. Tile Rendering Model (Informative)

### B.1 Scope and Intent

This appendix describes how **tile-based rendering** can be expressed naturally within the XPHMG execution model.

XPHMG **does not define a fixed graphics pipeline**, nor does it introduce an implicit ??otile renderer.???
Instead, tile rendering is treated as a **software-defined execution pattern** composed of:

* Explicit memory placement (via `CAP.XMEM.*` and `XPHMG_MEMCTL`),
* Explicit execution domains (RSV / MTX / GFX / RT),
* Explicit synchronization and visibility rules.

This model applies to **modern rendering and compute workloads**, including forward, deferred, hybrid, and visibility-driven pipelines.

No additional instructions are required to support tile rendering.

### B.2 Definition of a Tile in XPHMG

In XPHMG, a **tile** is not an architectural object.
It is a **logical working set** defined by software and characterized by:

* Residency in **LDS / XMEM Tier-0 or Tier-1** memory,
* A bounded spatial extent (e.g., screen-space region, cluster of primitives, or visibility batch),
* Local reuse across multiple execution phases.

The size, shape, and contents of a tile are entirely software-controlled.

### B.3 Memory Placement and Residency

Tile rendering relies on predictable locality.
XPHMG provides this via explicit memory control.

#### B.3.1 Recommended Memory Configuration

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

### B.4 Tile Rendering Phases (Conceptual)

Tile rendering in XPHMG is expressed as a sequence of **explicit phases**.
These phases are **conceptual**, not architectural requirements.

#### B.4.1 Tile Setup / Binning Phase

Purpose:

* Determine which primitives, fragments, or work-items belong to a tile.

Typical execution:

* RSV for compaction, prefix sums, index generation,
* MTX for transforms or small matrix operations,
* Writes to tile-resident lists or buffers in LDS/XMEM.

All data placement and ordering are explicit.

#### B.4.2 Tile Shading Phase

Purpose:

* Perform depth testing, sampling, shading, and local composition.

Typical execution:

* GFX operations (`ZTEST`, sampling, interpolation, shading helpers),
* RSV for per-lane or per-fragment control,
* Optional MTX for small MLPs, denoisers, or transforms.

All reads and writes operate on tile-resident data.

Hierarchical Z or early visibility mechanisms are expressed explicitly using GFX and memory ordering primitives.

#### B.4.3 Tile Resolve Phase

Purpose:

* Commit tile results to higher tiers of memory.

Typical execution:

* Explicit `XMEM.STG` or streaming stores,
* Optional compression or format conversion,
* Explicit fences to guarantee visibility.

No implicit resolve or flush is performed by hardware.

### B.5 Synchronization and Ordering

XPHMG requires **explicit synchronization** between tile phases.

Ordering guarantees are established using:

* LDS barriers (`XLDS.BAR`),
* Cache control and fences (`XMEM.CCTL`, `XFENCE.*`),
* Domain and class visibility rules from `CAP.XMEM.*`.

No execution phase implicitly synchronizes with another.

### B.6 Layout and Data Interoperability

Tile-local data may be stored using any layout expressible by the XPHMG Data Layout Model (XDL), including:

* Linear arrays,
* Canonical `XDL_TILE4x4_ROWMAJOR` tiles,
* Composite tiles,
* Engine-specific structured records.

When tiles are shared across execution domains, layouts must be interpreted consistently according to XDL rules.

### B.7 Determinism and Debug Visibility

Tile rendering in XPHMG is **fully deterministic** given:

* An identical instruction stream,
* Identical `CAP.PREC.*` and `CAP.XMEM.*` state,
* Identical descriptor parameters where used.

All state affecting execution is architecturally visible.
No hidden driver heuristics or implicit scheduling stages are permitted.

### B.8 Relationship to Immediate and Hybrid Rendering

The tile rendering model described here is **not exclusive**.

The same XPHMG implementation may support:

* Immediate-mode rendering,
* Tile-based rendering,
* Hybrid visibility or compute-driven pipelines,

without requiring ISA changes.

Tile rendering is an **execution strategy**, not a hardware mode.

### B.9 Summary

XPHMG supports tile rendering by design through:

* Explicit memory residency and classes,
* Explicit execution phases,
* Explicit synchronization,
* Explicit data layout rules.

This approach avoids implicit pipelines and opaque driver behavior while enabling efficient, modern tile-based rendering and compute workflows.

---

*This overview is non-normative; individual extension documents (`XPHMG_CAP`, `XPHMG_XMEM`, `XPHMG_RSV`, etc.) remain the normative source for instruction encodings and CSR bit layouts.*

---
**License:** CC-BY-SA 4.0 - Open Specification maintained by Pedro H. M. Garcia.
