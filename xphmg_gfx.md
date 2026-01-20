# RISC-V Vendor Extension **XPHMG_GFX**

**Graphics and Texture Pipeline Extension**

**Category:** Vendor Extension (`XPHMG_GFX`)
**Version:** 0.1.1
**Author:** Pedro H. M. Garcia
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview (Informative)

`XPHMG_GFX` defines a compact, instruction-level graphics and texture execution layer built on the XPHMG substrate. This section states the scope and role of the extension so implementers and toolchains can distinguish architectural behavior from CAP/XMEM/RSV/MTX/RT inheritance and from intentionally unspecified items in this revision.

The extension supplies primitives for texture and tensor sampling, interpolation and shading helpers, visibility and culling operations, depth testing and hierarchical Z, and programmable gathers and color transforms. It targets graphics-style and imaging workloads without mandating a fixed hardware pipeline.

`XPHMG_GFX` does not define a fixed pipeline. Coordinate transforms, control flow, batching, and memory orchestration are expressed explicitly using RSV, MTX, and XMEM. Software composes these primitives and enforces ordering with the fence and memory semantics defined later.

All numeric and memory behavior is taken from the effective state of `XPHMG_CAP`. Precision, rounding, saturation, NaN policy, accumulator promotion, masking (ZMODE), and trap enable come from CAP effective state. Memory ordering, coherence domain, class selection, streaming detection, descriptor policy, and SVMEM behavior come from `CAP.XMEM.*` and `CAP.XMEM.SVM*`. Any divergence from CAP/XMEM semantics is non-conforming. Effective predication consumed by GFX MUST be sourced only from `XPHMG_PMASK` as defined in `xphmg.md`; GFX does not define internal mask sources.

Architectural visibility: results of GFX operations, predicate masks (PMASK bank state), and CSR-visible status bits are architectural. Internal caching, interpolation microarchitecture, or internal promotion beyond CAP requirements is allowed only if architecturally visible results and exception signaling remain identical. NOTE: Implementations MAY promote internally for quality provided architectural results and traps/stickies are unchanged.

Interactions: GFX instructions rely on RSV window conventions for operands and results, on PMASK for predication, on MTX for transforms, on CAP for numeric semantics, and on XMEM for memory semantics. Optional RT use is permitted. Implementations MUST snapshot CAP/XMEM state at instruction boundaries as defined in the numeric and memory model; mid-instruction CSR changes MUST NOT be observed. No GFX instruction modifies CAP state.

---

## 2. Design Goals and Scope (Informative)

This section states the intended qualities, dependencies, and limits of `XPHMG_GFX`. Goals are informative; normative rules appear in later sections.

- Minimal instruction footprint: single instructions perform multi-step work (fetch, filter, convert) to reduce code size and dispatch, while remaining compatible with CAP precision and memory rules.
- Unified numeric control via `CAP.PREC.*`: all numeric policy (precision selection, accumulator width, rounding, saturation, NaN handling, ZMODE, trap enable) is inherited from CAP effective state; no GFX-specific numeric policy exists.
- CAP-driven memory defaults: coherence domain, memory class selection, streaming profiles, descriptor policy, and SVMEM behavior come from `CAP.XMEM.*` / `CAP.XMEM.SVM*`, consistent with RSV, MTX, and RT.
- LMEM-first execution with early culling: LMEM is the preferred working-set location; early elimination uses effective predication (PMASK) and culling to reduce external traffic.
- Predictable DMA/fence alignment with XMEM: ordering and visibility use `GFX.FENCE` and XMEM semantics so software can reason about completion and coherence across sampling, shading, and Z.
- Optional feature hooks: programmable filters, LUT color transforms, and hierarchical Z are provided, but no fixed pipeline stages are mandated; software composes them explicitly with RSV/MTX/XMEM control.

Architectural guarantees are limited to results, masks, and exceptions defined by the instruction and CSR semantics under CAP/XMEM. Microarchitectural optimizations that do not change architectural results (e.g., internal promotion) are allowed. Rasterization, triangle setup, and fixed-function blending are out of scope and, if needed, must be built in software with the provided primitives.

---

## 3. Dependencies and Profile Assumptions

This section lists the architectural prerequisites for a conforming `XPHMG_GFX` implementation. Numeric and memory semantics are inherited from CAP/XMEM; GFX does not redefine them.

**Mandatory:** These extensions and their contracts MUST be present in the same privilege context that executes GFX instructions.

- `XPHMG_CAP`: numeric and memory policy (`CAP.PREC.*`, `CAP.XMEM.*`, `CAP.XMEM.SVM*`). All GFX precision selection, accumulation, NaN policy, ZMODE, exception enables, coherence domain, memory class selection, streaming detection, descriptor defaults, and SVMEM behavior are inherited from CAP effective state.
- `XPHMG_PMASK`: Predicate state and banked predication. GFX consumes effective predication from PMASK and does not define predicate storage.
- `XPHMG_RSV`: Simple Vector windows. Operand/result placement uses RSV windows; predication is sourced from PMASK and masked-lane writeback follows CAP ZMODE.
- `XPHMG_MTX`: matrix multiply/transform ops. Culling and interpolation flows assume MTX is available for coordinate/vertex transforms when invoked by software.
- `XPHMG_XMEM`: DMA/LDS/descriptor plumbing via CAP state. All GFX memory accesses (including descriptor reads) rely on XMEM semantics for coherence, streaming profiles, compression defaults, and descriptor policy.

**Optional:** `XPHMG_RT` (shading path), descriptor-based offload frameworks built on `XPHMG_MEMCTL`. GFX does not require RT or MEMCTL; absence does not change GFX semantics.

`XPHMG_GFX` does not restate CAP fields; it consumes the effective CAP state latched at the instruction boundary.

Profile assumptions: CAP, PMASK, RSV, MTX, and XMEM CSRs are expected to be accessible and effective in the same privilege level as GFX CSRs. If the CAP window `0x7C0-0x7FF` is reserved, GFX CSRs MUST be placed in a non-overlapping vendor window while preserving field semantics.

Architectural visibility: dependencies on CAP/PMASK/RSV/MTX/XMEM are architectural. Optional RT or MEMCTL offload does not change GFX-defined results; such components only provide additional software-managed control.

TODO: Document privilege/profile assumptions and CSR window placement/remap policy.
TODO: Specify CSR window placement/remap policy per profile.

---

## 4. Architectural State

This section defines the architectural state specific to `XPHMG_GFX`, limited to the vendor CSR window used by GFX instructions. All fields are architecturally visible and consumed by GFX operations. Implementations MAY cache or pipeline these values internally but MUST preserve architectural results, reset behavior, and visibility rules. CAP/XMEM remain authoritative for numeric and memory semantics.

### 4.1 CSR Map and Field Summaries (GFX window)

The table below lists GFX CSRs, their addresses, and the function of each field. Numeric interpretation, NaN policy, saturation, quantization, and memory policy are derived from CAP effective state; these CSRs provide configuration inputs interpreted under CAP/XMEM rules. If a platform relocates the window to avoid CAP (`0x7C0-0x7FF`), it MUST preserve all field semantics and effective behavior.

| Addr          | Name         | Description                                                                                                       |
| ------------- | ------------ | ----------------------------------------------------------------------------------------------------------------- |
| **0x800**     | `GFXCFG`     | Global flags and format hints: `EN`, `CONVERT_EN`, `FILTER_DFLT`, `SRGB_EN`, `WRAP_DFLT`, `TEX_FMT_HINT`, `NORM`. |
| **0x801**     | `GFXWRAP`    | Wrapping mode: `00=clamp`, `01=repeat`, `10=mirror`, `11=border`.                                                 |
| **0x802**     | `GFXTEXBASE` | Base pointer (LMEM or DRAM).                                                                                      |
| **0x803**     | `GFXTEXPITCH`| Bytes per row.                                                                                                    |
| **0x804-0x809** | `GFXPLN0..5`| Frustum planes in clip space, each `(A,B,C,D)` for `Ax+By+Cz+Dw <= 0`.                                            |
| **0x80A**     | `GFXTEXWH`   | Texture width/height.                                                                                             |
| **0x80B**     | `GFXTEXFMT`  | Format enum (UNORM8, FP16, BF16, INT8x4, etc.) + swizzle hints.                                                   |
| **0x80C**     | `GFXBORDERC` | Border color (PET/EW-compliant).                                                                                  |
| **0x80D**     | `GFXCLIP_EPS`| Epsilon used for `w <= eps` reject in culling ops.                                                                |
| **0x80E**     | `GFXZCFG`    | Z test config: comparison op, write enable, early/late mode, reverse-Z flag.                                      |
| **0x80F**     | `GFXZBASE`   | Depth buffer base pointer (LMEM/DRAM).                                                                            |
| **0x810**     | `GFXZPITCH`  | Bytes per row in Z buffer.                                                                                        |
| **0x811**     | `GFXZWH`     | Z buffer dimensions.                                                                                              |
| **0x812**     | `GFXHIZCFG`  | Hierarchical-Z config: tile width/height, level count.                                                            |
| **0x813**     | `GFXLUTBASE` | Base pointer for 1D/3D LUT tables.                                                                                |
| **0x814**     | `GFXLUTFMT`  | LUT format and dimensions.                                                                                        |
| **0x815**     | `GFXGATHBASE`| Base pointer for programmable gather offsets.                                                                     |
| **0x816**     | `GFXPEERHINT`| Peer LMEM bus hint: `{peer_id, qos, prefetch, non_temporal}`.                                                     |
| **0x817**     | `GFXTEXSTRIDE1` | Next-dimension stride for tensor layouts (e.g., channel/batch).                                                 |

> All CSR numeric and memory behavior is governed by CAP. Address placement MUST avoid the CAP window `0x7C0-0x7FF`; if occupied, remap GFX CSRs to a vendor-private block while preserving field semantics.

**GFXZCFG (0x80E) bit fields**  
- bits[2:0]  CMP: 0=LESS,1=LEQUAL,2=GREATER,3=GEQUAL,4=EQUAL,5=NEQUAL,6=ALWAYS,7=NEVER  
- bit[3]     WRITE_EN (ZWRITE implied when 1 and op permits)  
- bit[4]     EARLY_EN (enable early-Z in `GFX.ZTEST e=1`)  
- bit[5]     REVERSE_Z (invert comparison sense for CMP in hardware)  
- bits[31:6] reserved=0

**GFXHIZCFG (0x812) bit fields**  
- bits[3:0]   TILE_W_LOG2 (e.g., 3->8px, 4->16px)  
- bits[7:4]   TILE_H_LOG2  
- bits[15:8]  LEVELS (number of HZ mip levels)  
- bits[31:16] reserved=0

**GFXLUTFMT (0x814) bit fields**  
- bits[1:0]   DIM: 0=1D, 1=3D  
- bit[2]      NORM_IDX (1=normalized indices)  
- bits[7:4]   SIZE_R (log2 or direct size per implementer; document)  
- bits[11:8]  SIZE_G  
- bits[15:12] SIZE_B  
- bits[31:16] reserved

### 4.2 CSR Access/Privilege/Reset Requirements

Access model: GFX CSRs are vendor CSRs and MUST be accessible in the privilege level that executes GFX instructions. Implementations MUST document whether lower privilege levels MAY read or write these CSRs; unauthorized accesses MUST follow platform trap or deny rules. CSR writes take effect at the next instruction boundary; in-flight instructions observe state latched at decode.

Reset behavior: Implementations MUST define reset values for all GFX CSRs. `GFXCFG.EN` SHOULD reset to disabled unless a profile mandates enabled-by-default behavior; other fields SHOULD reset to neutral defaults (e.g., wrap=clamp, filter=nearest, format hints cleared) consistent with CAP defaults. NOTE: Exact reset values are platform-specific; v0.1 does not mandate numeric defaults beyond disabled state.

Width and alignment: CSRs are 32-bit unless the platform profile defines 64-bit CSR access. CSRs that hold addresses or 64-bit quantities (`GFXTEXBASE`, `GFXZBASE`, `GFXLUTBASE`, `GFXGATHBASE`) MUST follow the platform’s CSR width convention (single 64-bit register or paired 32-bit registers). Software MUST use the platform-defined access sequence to avoid torn reads/writes. TODO: Clarify CSR width convention for 64-bit addresses in target profiles.

Remapping and window placement: If the CAP window occupies `0x7C0-0x7FF`, GFX CSRs MUST reside in a non-overlapping vendor block while preserving field semantics. Software targeting multiple profiles SHOULD treat the GFX CSR base as platform-provided. TODO-VERIFY: Confirm mapping relative to `0x7C0-0x7FF` CAP window for each profile.

---

## 5. Programming Model and Conventions

This section summarizes programming conventions for `XPHMG_GFX`: how operands are supplied, how coordinates and descriptors are interpreted, and how RSV windows are used. It is informative and relies on CAP/XMEM for numeric and memory semantics. These conventions do not override the normative numeric and memory rules.

Descriptor versus CSR precedence: When a descriptor pointer and CSRs provide equivalent fields (base, pitch, dimensions, format, wrap, border), the descriptor overrides the corresponding CSRs for that instruction. Fields not present in the descriptor fall back to CSRs, which are interpreted under CAP/XMEM policy. This precedence is architectural. NOTE: Malformed descriptor content or CAP-inconsistent settings trigger the exception behavior defined in the numeric/memory model and exceptions sections. TODO: Define platform-specific descriptor validation requirements if needed.

RSV windows: GFX instructions consume and produce RSV windows as listed below. Effective predication is sourced from PMASK; masked-lane writeback follows CAP ZMODE (merge or zero). Predicate updates, when supported, write the selected PMASK bank and are architectural. Software MUST ensure window contents conform to CAP precision and formatting expectations before invoking GFX instructions.

### 5.1 Sampling & Coordinate Conventions

This subsection defines interpretation of sampling coordinates, wrapping, normalization, and descriptors. Numeric interpretation of coordinates and fetched data follows CAP precision and NaN/saturation rules; memory behavior follows CAP.XMEM.

- Coordinates are normalized `[0,1)` when `GFXCFG.NORM=1`; otherwise texel-space.
- Texel centers are at `i+0.5`.
- Bilinear filtering uses `fract(u), fract(v)`.
- Out-of-range handling follows wrapping policy; `border` uses `GFXBORDERC`.
- sRGB conversion is applied only when both `f=1` and `SRGB_EN=1`.
- Descriptors: If `rs1` points to a `TextureDesc` (`base,pitch,w,h,fmt,border`), it overrides CSRs.

**TextureDesc (memory-resident) layout (little-endian, 16-byte aligned):**
```c
struct TextureDesc {
    u64 base;       // LMEM/DRAM base address
    u32 pitch;      // bytes per row (stride 0)
    u32 w, h;       // dimensions in texels
    u32 fmt;        // matches GFXTEXFMT enum (channel type/width, swizzle)
    u32 wrap;       // 0=clamp, 1=repeat, 2=mirror, 3=border
    u32 border[4];  // RGBA (PET/EW-compliant packing) or fp32 if PET is fp*
    u32 flags;      // bit0: normalized coords, bit1: sRGB, bit2: planar, etc.
    // optional tensor strides (when layout != 2D):
    u32 stride1;    // next dimension (e.g., channel or row-major plane)
    u32 stride2;    // next dimension (batch or feature map)
};
```
If `rs1` points to `TextureDesc`, it overrides equivalent CSRs for that instruction. Descriptor memory accesses use CAP XMEM defaults (class, domain, streaming profile, compression, descriptor policy).

### 5.2 RSV Window Conventions

This subsection assigns RSV windows to operands and results; predicate outputs, when specified, update PMASK banks and are not RSV window state. The mapping is architectural. Predicate-false lanes follow CAP ZMODE for all architecturally visible outputs (register state or memory writes).

| Instruction  | Input Windows      | Output Windows | Notes                    |
| ------------ | ------------------ | -------------- | ------------------------ |
| `GFX.SAMPLE` | `RSV a0..a1` (u,v) | `RSV r0..r3`   | RGBA sample              |
| `GFX.INTERP` | `r0..r3`, `a2..a3` | `r4..r7`       | Bilinear/perspective     |
| `GFX.SHADE`  | `r*`, `n*`, `l*`   | `r*`           | Lighting                 |
| `GFX.ZTEST`  | `rZ`               | PMASK bank     | Predicate result (optional update) |
| `GFX.LUT*`   | `rRGB`             | `rRGB`         | In-place color transform |

ZMODE from CAP applies to predicate-false lanes for all operations (see section 6.3).

---

## 6. Numeric and Memory Model (CAP-Driven Contract)

This section defines the numeric and memory contract for `XPHMG_GFX` as derived from CAP and XMEM effective state. GFX introduces no independent numeric or memory policy; every operation MUST apply CAP and XMEM rules uniformly across sampler, gather, interpolation, shading, Z pipeline, LUT, culling, and compaction units.

### 6.1 Precision/Accumulation/NaN/ZMODE Rules

All GFX units consume only the effective state from `CAP.PREC.*`. The following rules apply uniformly:

- **PET/EW/ACCW selection:** Derived from `CAP.PREC.MODE/ALT` at the instruction boundary. A per-op `ppp` requests a mapping; the effective PET/EW/ACCW is resolved under CAP policy. If the requested mapping cannot be realized, the operation traps as Unsupported and sets `CAP.PREC.STAT.UNSUP_FMT`.
- **Rounding, saturation, quantization, NaN, ZMODE, SAE, traps:** Follow CAP effective state (`CAP.PREC.STAT`, `CAP.PREC.EXC.EN`). No GFX-specific numeric policy exists.
- **Accumulator policy:** If CAP effective ACCW indicates promotion (e.g., ACCW=FP32 or auto), GFX units MUST promote internal accumulation accordingly. If a fixed ACCW is insufficient, software MUST select a suitable CAP mode; GFX does not override ACCW.
- **Masked lanes:** Predicate-false lanes are defined by PMASK effective predication; writeback is governed by `CAP.PREC.MODE.ZMODE` (merge vs zero) as specified in section 6.3.
- **Exceptions and stickies:** FP/precision stickies are reported in `CAP.PREC.EXC.ST` and `CAP.PREC.STAT`; traps occur only if enabled and effective SAE is 0. NaN propagation/squash follows `CAP.PREC.MODE.NAN_POL`. Color-space conversion MUST honor CAP NaN/FTZ/quantization rules.
- **Internal promotion latitude:** Implementations MAY promote internally (e.g., to FP32) provided outputs and exception signaling match CAP-defined results. Any divergence is non-conforming.

**Processing order (informative):** `fetch/convert -> filter/interp/gather -> color-space convert -> pack/saturate` reflects expected sequencing for applying CAP numeric rules.

### 6.2 Memory Semantics via CAP.XMEM/SVMEM

All GFX memory accesses (texture fetch, Z/LUT/gather loads/stores, descriptor reads, compaction writes) follow `CAP.XMEM.*` and `CAP.XMEM.SVM*`. GFX introduces no divergent memory policy; overrides MAY come only from descriptor fields that explicitly replace a CAP default.

- **Coherence and domain:** Obey `CAP.XMEM.DOM` for all GFX memory operations. Cross-core visibility follows the configured domain.
- **Memory class and QoS:** Use class mapping from `CAP.XMEM.CLSMAP` and defaults from `CAP.XMEM.DESCPOL`. Descriptor overrides apply only to the consuming instruction.
- **Streaming and compression:** Apply streaming detection and profiles per `CAP.XMEM.STREAM_CTL` and `CAP.XMEM.SPROF*`. Compression defaults follow `CAP.XMEM.COMP_CTL`; descriptors may override if they carry MEMCTL trailers.
- **LDS budgets:** Respect `CAP.XMEM.LDS_BUDGET` for local memory usage. GFX MUST NOT exceed budgets assumed by the CAP profile.
- **SVMEM behavior:** Use `CAP.XMEM.SVM*` rules for all indexed gather/scatter paths, including masked-lane handling and FOF behavior.
- **Descriptor scope:** Descriptors override only fields they contain. Unspecified fields fall back to CAP defaults. Invalid or unsupported descriptor combinations MUST trap or set status per CAP/XMEM rules.

Predicate-false lanes MUST NOT issue memory accesses for any GFX operation that touches memory; this rule is defined by `xphmg_xmem.md` and applies uniformly to vertex/attribute fetch, texture sampling, LUT/Z/gather access, descriptor reads, and any buffer/image/atomic paths.

### 6.3 Predication and Masked Execution (PMASK-Driven)

GFX consumes only the effective predication defined by PMASK:

```
pred[i] = PMASK.bank[pbank_eff][i]
```

`pbank_eff` is selected by an instruction field where present, or by an existing architectural predication prefix; if no selection is made, `pbank=0` is implied and behaves as a virtual ALL-ONES bank.

GFX MUST NOT redefine PMASK, introduce GFX-specific predicate CSRs, or define alternative predication modes; it is strictly a consumer of PMASK (predication), CAP (writeback policy), and XMEM (memory semantics) as defined in `xphmg.md`.

Predicate-false lanes or fragments (`pred[i]=0`) MUST NOT execute, MUST NOT read attributes, vertices, parameters, or interpolants, MUST NOT raise exceptions, and MUST NOT produce side-effects.

Writeback for predicate-false lanes is governed exclusively by `CAP.PREC.MODE.ZMODE`:
- `merge`: preserve the previous destination value.
- `zero`: write architectural zero.
GFX MUST NOT define any other writeback policy under predication.

Predicate-producing GFX operations, when specified by the instruction, write results to `PMASK.bank[pbank_eff]`. Writes to `pbank=0` have no effect.

### 6.4 State Latching at Instruction Boundary

Each GFX instruction MUST snapshot at decode:

- `CAP.PREC.MODE`
- `CAP.PREC.ALT`
- `CAP.PREC.EXC.EN`
- Relevant `CAP.XMEM.*` and `CAP.XMEM.SVM*` fields (coherence domain, memory classes, streaming profiles, LDS budgets, compression defaults, descriptor defaults, SVMEM parameters)

Mid-instruction CSR changes MUST NOT affect an in-flight operation. The effective state observed is the state latched at decode; software relying on CAP/XMEM updates MUST order them before issuing dependent GFX instructions, using fences where needed.

---

## 7. Instruction Encodings and Formats

This section specifies the instruction encoding patterns used by `XPHMG_GFX`. All instructions share major opcode `0x2B` and use I-type layouts with opcode-dependent immediate and `funct3` subfields. Encodings define placement of operands, immediates, and precision selector bits within the 32-bit instruction word. Numerical and memory behavior remains governed by CAP/XMEM effective state; encoding fields do not modify CAP state.

### 7.1 Common I-Type Encoding and Precision Selector Rules

All GFX instructions use the 32-bit I-type encoding with opcode `00101011b` (0x2B):  
`imm[11:0] | rs1 | funct3 | rd | 00101011b`.

Operand fields: `rd` is the destination register, `rs1` supplies the primary operand (often a pointer or coordinate pair), and `funct3` selects the instruction class. Immediate bits are subdivided per instruction to carry precision selectors (`ppp`) and operation-specific flags.

Precision selector (`ppp`): Encodings that include `ppp` request a precision mapping. `ppp=111` selects the CAP effective state latched at decode. Other `ppp` values are advisory and must be reconciled with CAP policy; if the requested PET/EW cannot be realized, the instruction traps as Unsupported and sets `CAP.PREC.STAT.UNSUP_FMT`. The encoding does not modify CAP state.

Optional ALT selector: Some encodings may reserve an immediate bit as `ALT` to request `CAP.PREC.ALT` instead of `CAP.PREC.MODE`. `ALT=0` selects MODE, `ALT=1` selects ALT merged effective state. If the encoding omits an ALT bit, bank choice follows `CAP.PREC.ALT` policy only.

Other immediate fields: Instruction-specific immediate bits (e.g., filtering mode, wrap override, tap count) are interpreted per instruction and do not alter CAP state. They are applied subject to CAP numeric and memory rules.

Z value format and width are governed by CAP effective precision; compare, reverse-Z, and early/late mode are configured via `GFXZCFG`. Encoding fields that reference Z behavior request these modes but cannot override CAP precision.

### 7.2 Encoding Table (Summary)

The table below summarizes encoding patterns for all GFX instructions, including immediate subfields and `funct3` values. It complements the per-instruction semantics; implementers MUST apply CAP/XMEM rules when decoding precision and memory behavior.

| Instruction         | I-type fields   | funct3 | Opcode     | Description                 |
|---------------------|-----------------|--------|------------|-----------------------------|
| `GFX.SAMPLE`        | `ppp m f w`     | `010`  | `00101011` | Texture/tensor sampling     |
| `GFX.SAMPLE.GATH`   | `ppp n p s`     | `010`  | `00101011` | Programmable gather pattern |
| `GFX.INTERP`        | `ppp M P B`     | `011`  | `00101011` | Interpolation               |
| `GFX.SHADE`         | `ppp OO L N`    | `100`  | `00101011` | Arithmetic shading          |
| `GFX.ZTEST`         | `ppp w e m`     | `101`  | `00101011` | Depth test                  |
| `GFX.ZWRITE`        | `fixed`         | `101`  | `00101011` | Z write                     |
| `GFX.HZ.BUILD`      | `ppp tt ll hh`  | `001`  | `00101011` | Hierarchical-Z reduce       |
| `GFX.LUT1D/3D`      | `ppp d i c`     | `100`  | `00101011` | LUT transform               |
| `GFX.FENCE`         | `0000 00 mm ff` | `101`  | `00101011` | Pipeline barrier            |
| `GFX.CULL.*`        | (see section 8.7) | varies | `00101011` | Culling/compaction ops      |

---

## 8. Instruction Semantics

This section defines normative semantics for all `XPHMG_GFX` instructions. Each instruction assumes the numeric and memory contract from section 6, descriptor/CSR precedence from section 5, and PMASK-sourced predication with CAP ZMODE writeback (section 6.3). Implementation latitude is limited to microarchitectural choices that do not alter architecturally visible results, masks, or exception signaling. Operand/result placement follows the RSV window conventions in section 5; all memory accesses follow CAP.XMEM rules.

### 8.1 Sampling and Gather

These instructions fetch and optionally filter samples from textures or tensors. Coordinates, wrapping, filtering, and color-space controls come from immediates and GFX CSRs or descriptors; numeric interpretation and memory policy are defined by CAP/XMEM. Predicate-false lanes are suppressed per section 6.3; writeback follows CAP ZMODE for all outputs, including partial gathers.

#### `GFX.SAMPLE` -- Texture/tensor sampling

```
ppp m f w | xxxxx | 010 | xxxxx | 00101011
```

- `ppp`: precision selector (CAP effective state).
- `m`: filtering (0 = nearest, 1 = bilinear).
- `f`: color-space (0 = linear, 1 = sRGB->linear if enabled).
- `w`: wrapping override (0 = use CSR, 1 = use imm[3:2]).

**Semantics:**  
`rd <- SAMPLE(TEX, coords from rs1, base/pitch from CSRs or descriptor)`.

- Accepts CSR-bound or pointer-to-descriptor mode.
- Coordinates follow normalized `[0,1)` or texel-space depending on `GFXCFG.NORM`.
- Fractional parts are used for bilinear filtering.
- Out-of-range obeys `GFXWRAP`; `border` returns `GFXBORDERC`.
- If `f=1` and `SRGB_EN=1`, applies sRGB->linear post-fetch.

#### `GFX.SAMPLE.GATH` -- Programmable gather pattern

```
ppp n p s | xxxxx | 010 | xxxxx | 00101011
```

- `ppp`: precision (CAP effective state).
- `n`: number of taps (from imm or descriptor).
- `p`: weighted accumulate (0=average, 1=use weights from table).
- `s`: signed offset mode.

**Semantics:**  
Gathers `n` texels according to the offset table in `GFXGATHBASE` (entries `{du, dv, weight}`). Results accumulate in the effective accumulator width (CAP-driven) and are down-converted on writeback under CAP rules (SAT, NAN_POL, FP_RMODE, quantization).

**Offset table format (at `GFXGATHBASE`)**  
Each tap entry is `{du, dv, weight}` with PET/EW from CAP. For integer PET, `du/dv` use fixed-point per EW; for floating PET, they are FP. Maximum taps `n` is implementation-defined but MUST be >= 8; values above the limit MAY be truncated (setting `CAP.PREC.STAT.UNSUP_FMT` is permitted).

### 8.2 Interpolation

Interpolation combines nearby samples based on fractional coordinates. All arithmetic, including optional bias and perspective division, follows CAP numeric policy. Inputs come from RSV; predicate-false lanes are suppressed per section 6.3 and writeback follows CAP ZMODE.

#### `GFX.INTERP` -- Bilinear/perspective interpolation

```
ppp M P B | xxxxx | 011 | xxxxx | 00101011
```

- `M`: 0 = bilinear, 1 = perspective.
- `P`: premultiplied-alpha variant.
- `B`: bias enable (adds bias post-interp).

**Semantics:**  
`rd <- interp2D(tex00, tex01, tex10, tex11, ufrac, vfrac [,bias])`.

Inputs and outputs follow CAP precision; accumulation uses CAP effective ACCW with CAP rounding/NAN_POL.

### 8.3 Shading

Shading operations consume RSV source windows (including optional normal/light windows) and produce RSV results. Numeric behavior, including normalization and dot products, follows CAP effective state; predicate-false lanes are suppressed per section 6.3 and writeback follows CAP ZMODE.

#### `GFX.SHADE` -- Arithmetic shading / lighting combine

```
ppp OO L N | xxxxx | 100 | xxxxx | 00101011
```

- `OO`: op (`00=add`, `01=sub`, `10=mul`, `11=dot`).
- `L`: lighting enable (uses normal/light RSV banks).
- `N`: normalize result.

**Semantics:**  
`rd <- shade(op, srcA, srcB [,normal/light])`, honoring CAP precision, rounding, saturation, and quantization settings.

### 8.4 Depth/Z Pipeline

Depth test, depth write, and hierarchical-Z build operations interact with Z buffers in LMEM/DRAM using CAP.XMEM semantics and CAP precision. Atomicity and ordering are defined per instruction and CAP/XMEM policy; predicate-false lanes are suppressed per section 6.3.

#### `GFX.ZTEST` -- Depth comparison

```
ppp w e m | xxxxx | 101 | xxxxx | 00101011
```

- `w`: write enable (0=test only, 1=update).
- `e`: early-Z mode flag (test before shading).
- `m`: mask update policy (0=replace, 1=mask&=pass).

**Semantics:**  
Compares `z_in` from RSV or LMEM with `Zbuffer` (per `GFXZCFG.CMP`). Writes new Z if passed and `w=1`. Returns per-lane predicate result; optionally updates the selected PMASK bank. Numeric format and NaN handling follow CAP effective precision and `CAP.PREC.MODE.NAN_POL`.

**Atomicity & ordering:**  
If `w=1`, `GFX.ZTEST` performs a compare-and-conditional-write that is atomic per pixel within the core. Visibility to LMEM/DRAM follows the fence scope (see `GFX.FENCE`). Cross-core atomicity is guaranteed only when the Z buffer resides in LMEM with single-writer ownership or when the fabric provides atomic writes for the configured width.

#### `GFX.ZWRITE` -- Write Z-buffer (explicit)

```
0000 00 01 00 | xxxxx | 101 | xxxxx | 00101011
```

Writes per-lane depth values (from RSV) to `Zbuffer`, respecting pitch and CAP precision. Does not recheck CMP; use `GFX.ZTEST` for guarded updates.

#### `GFX.HZ.BUILD` -- Hierarchical-Z reduce

```
ppp tt ll hh | xxxxx | 001 | xxxxx | 00101011
```

- `tt`: reduction type (`00=min`, `01=max`).
- `ll`: level of hierarchy to build/update.
- `hh`: tile height/width exponent (log2).

**Semantics:**  
Reduces Zbuffer tiles into a coarser hierarchy level in LMEM or DRAM, producing min/max depth per tile for early rejection. Internal accumulation MAY be promoted per CAP ACCW; outputs follow CAP PET/EW.

### 8.5 Color/LUT

LUT-based color transforms use CAP.XMEM for table access and CAP numeric rules for conversion/interpolation. Predicate-false lanes are suppressed per section 6.3; writeback follows CAP ZMODE for in-place transforms.

#### `GFX.LUT1D` / `GFX.LUT3D` -- Color LUT transform

```
ppp d i c | xxxxx | 100 | xxxxx | 00101011
```

- `d`: LUT dimension (0=1D, 1=3D).
- `i`: interpolation (0=nearest, 1=linear/trilinear).
- `c`: channel mode (0=RGB, 1=RGBA or custom).

**Semantics:**  
Applies color transform from LUT table in `GFXLUTBASE`/`GFXLUTFMT`. Uses normalized indices (0..1) or texel-space indices depending on CSR. Results are written back to RSV, respecting CAP precision, rounding, NaN, and saturation rules.

### 8.6 Fences

Pipeline barriers order GFX units and related memory operations. Fences do not modify data; they enforce completion and visibility per the selected scope and mask using XMEM semantics. They do not alter CAP state.

#### `GFX.FENCE` -- Pipeline barrier

```
0000 00 mm ff | 00000 | 101 | 00000 | 00101011
```

- `ff`: scope (`00=local`, `01=LMEM`, `10=DRAM`, `11=global`).
- `mm`: mask (`00=sample`, `01=interp`, `10=shade`, `11=all`).

**Semantics:**  
Ensures completion/visibility across selected units.

| Scope  | Guarantee                                  |
| ------ | ------------------------------------------ |
| Local  | Orders operations within lane.             |
| LMEM   | Ensures LMEM ops (GFX + XMEM) are visible. |
| DRAM   | Waits on XMEM.DMA completion.              |
| Global | Orders across all peers/units.             |

`ff=DRAM` waits for completion of all XMEM.DMA transactions issued by the same core before the fence; it does not stall unrelated peers unless `ff=global`. Coherence domain follows `CAP.XMEM.DOM`.

### 8.7 Culling/Visibility Instructions

Per-lane visibility and compaction operations consume coordinates or bounding volumes from RSV, apply clip tests using GFX CSRs and CAP numeric rules, and update PMASK or memory according to section 6.3 and XMEM semantics. Mask updates are architecturally visible; implementations MUST preserve specified behavior.

#### `GFX.CULL.PLANE`

```
tt s k z | xxxxx | 110 | xxxxx | 00101011
```

Per-lane plane test (`Ax+By+Cz+Dw`).  
`tt`: comparison type, `k`: keep-policy, `z`: reject if `w<=GFXCLIP_EPS`.

#### `GFX.CULL.FRUSTUM`

```
tt _ k z | xxxxx | 111 | xxxxx | 00101011
```

AND of 6 planes (`GFXPLN0..5`).

#### `GFX.CULL.AABB`

```
ee f k x | xxxxx | 001 | xxxxx | 00101011
```

AABB vs frustum; transforms 8 corners via `MTX.MUL` if `f=1`.

#### `GFX.CULL.COMPOSE`

```
op inv _ _ | 00000 | 000 | 00000 | 00101011
```

Combines PMASK predicate masks (`AND/OR/XOR/COPY`).

#### `GFX.CULL.COMPACT`

```
mm aa rr cc | xxxxx | 000 | xxxxx | 00101011
```

Stream compaction; writes only surviving lanes to LMEM (indices and/or attributes). If `cc=1`, returns count in `rd`. Predicate-false lanes are suppressed per section 6.3; writeback follows CAP ZMODE.

**Clip W handling:** with `z=1`, lanes with `!isfinite(w)` or `w <= GFXCLIP_EPS` are rejected before plane tests.

---

## 9. Execution Model (Informative)

This section provides informative composition of `XPHMG_GFX` instructions. It adds no normative requirements beyond instruction semantics, numeric/memory model, and fence ordering rules. Software MUST enforce ordering or visibility with `GFX.FENCE` and CAP/XMEM facilities; implementations MUST honor guarantees defined by those mechanisms.

### 9.1 Typical Flow (2D/3D)

The sequence below illustrates a common ordering for 2D/3D workloads. It MAY be reordered if required ordering is enforced with fences and CAP/XMEM policy.

```
1. Transform vertices: MTX.MUL (model->clip)
2. Frustum culling: GFX.CULL.FRUSTUM
3. Optional AABB compaction: GFX.CULL.COMPACT
4. Early-Z (optional): GFX.ZTEST (e=1)
5. SAMPLE -> INTERP -> SHADE
6. Late-Z: GFX.ZWRITE (if enabled)
7. Color post-processing: GFX.LUT1D/3D
8. FENCE (LMEM or DRAM)
```

Architectural considerations: Early-Z MAY reduce work but MUST NOT change results relative to a late Z test when CAP precision and `GFXZCFG` settings are applied consistently. Fences are required to guarantee visibility to LMEM/DRAM for later stages or peers. If descriptors are used, they MUST be populated before issuing dependent GFX instructions and follow CAP.XMEM ordering.

### 9.2 Wavefront Queues & Hierarchical-Z (Informative)

Wavefront queues: Software-managed queues MAY be implemented as LMEM ring buffers. Survivors are enqueued with `GFX.CULL.COMPACT`. Queue accesses MUST follow CAP.XMEM class, streaming profile, coherence domain, and compression defaults unless overridden by descriptor policy. Ordering and visibility across producers/consumers MUST be enforced with fences appropriate to the queue’s memory class and domain.

Hierarchical Z: `GFX.HZ.BUILD` constructs coarser Z representations to enable early rejection. Tile fetches/stores for HZ levels use CAP.XMEM settings; numerical reduction follows CAP precision rules. Software MUST sequence HZ builds relative to subsequent `GFX.ZTEST` invocations using fences to ensure HZ data is visible at the desired scope.

NOTE: This execution model is illustrative; platforms MAY implement additional scheduling or offload mechanisms provided architectural results and ordering defined by GFX semantics and CAP/XMEM rules are preserved.

---

## 10. Interoperability and Implementation Requirements

This section defines interoperability expectations and implementation requirements relative to CAP, XMEM/SVMEM, and optional vendor facilities. It clarifies descriptor overrides, determinism across CSR updates, and advisory hint behavior. Requirements are normative unless marked advisory.

### 10.1 Descriptor Overrides and Limits

Descriptors MAY override only the fields they contain (base/pitch/format/wrap, optional MEMCTL block). Unspecified fields fall back to CAP defaults. If a descriptor includes an `XPHMG_MEMCTL` trailer, only `dom`, `mclass`, `sprof_id`, `hyb`, `sec` override CAP; other memory semantics remain bound to CAP.

Architectural scope: Descriptor overrides apply to the consuming instruction and do not modify CAP state. Unsupported or inconsistent combinations (e.g., unsupported PET/EW or memory class) MUST trap or set status per CAP/XMEM rules. Implementations MUST NOT silently coerce descriptors beyond CAP allowances. TODO: Specify descriptor length, alignment, and validation rules per profile.

### 10.2 Apply/Determinism Rules

- `CAP.PREC.MODE/APPLY0` and `CAP.PREC.ALT/APPLY1` take effect atomically; GFX MUST NOT observe partial writes.
- XMEM/SVMEM CSRs become effective at the next instruction boundary; GFX MUST latch them once per instruction.
- Debug/replay MUST see a stable effective state (mirror of `CAP.PREC.STAT` and current `CAP.XMEM.*`).

Determinism: GFX instructions MUST produce results consistent with the CAP/XMEM state latched at decode. Mid-instruction CSR updates MUST NOT affect in-flight operations. Software MUST order CAP/XMEM updates and dependent GFX instructions with fences as needed. NOTE: Behavior when CSR writes race with instruction fetch/decode is governed by platform CSR atomicity rules; GFX relies on those rules.

### 10.3 Peer LMEM Bus Hints (Informative/advisory)

`GFXPEERHINT` provides optional guidance to inter-tile or multi-core fabrics:

| Field          | Meaning                             |
| -------------- | ----------------------------------- |
| `peer_id`      | Target peer LMEM ID.                |
| `qos`          | Quality-of-service / priority hint. |
| `prefetch`     | Suggest prefetch size or stride.    |
| `non_temporal` | Disable caching if set.             |

Hints are advisory only; correctness MUST NOT depend on them.

Architectural visibility: `GFXPEERHINT` does not alter architectural results or memory ordering; it MAY influence microarchitectural QoS or path selection. Implementations MAY ignore these hints without affecting correctness.

---

## 11. Exceptions and Status

This section summarizes architecturally visible exceptions and status updates produced by `XPHMG_GFX` instructions. Exception signaling and sticky updates are governed by CAP precision and memory policy; GFX introduces no independent exception mechanisms. The table below lists conditions and resulting actions; implementations MUST report stickies and traps through CAP state as specified in the numeric and memory model.

| Condition                                                                          | Action                                                                                         | Source of control |
| ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ----------------- |
| Unsupported combination of PET/EW/ACC requested by instruction or CAP effective    | **Illegal Instruction** (or **Unsupported Feature**) and set `CAP.PREC.STAT.UNSUP_FMT=1`       | CAP               |
| Overflow/underflow/inexact/NaN per CAP effective precision                         | Update `CAP.PREC.EXC.ST` stickies (trap only if enabled in `CAP.PREC.EXC.EN` and SAE=0)        | CAP               |
| Masked-lane zero/merge mismatch                                                    | Behave per CAP ZMODE; any divergence is an architectural bug                                   | CAP               |
| Invalid LMEM/DRAM access                                                           | **Load/Store Access Fault**                                                                    | Memory subsystem  |
| GFX disabled (`GFXCFG.EN=0`)                                                       | No-op (or optional trap)                                                                       | GFXCFG            |

Sticky precision faults are visible in `CAP.PREC.STAT` when implemented. Trap delivery follows CAP trap enable and SAE rules; GFX instructions MUST NOT generate traps outside CAP-defined mechanisms. When `GFXCFG.EN=0`, implementations MAY treat GFX instructions as no-ops or raise a trap per platform policy; the platform MUST document this choice. TODO: Clarify platform-specific trap policy when `GFXCFG.EN=0` is cleared.

---

*This specification is part of the PHMG Open Specification Set (CC-BY-SA 4.0).* 
