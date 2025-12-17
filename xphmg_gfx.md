# RISC-V Vendor Extension **XPHMG_GFX**

**Graphics and Texture Pipeline Extension**

**Category:** Vendor Extension (`XPHMG_GFX`)
**Version:** 0.1.0
**Author:** Pedro H. M. Garcia
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview (Informative)

`XPHMG_GFX` defines a compact, instruction-level **graphics and texture execution layer** built on top of the XPHMG execution substrate.

The extension provides primitives for:

* Texture and tensor sampling,
* Interpolation and shading helpers,
* Visibility and culling operations,
* Depth testing and hierarchical Z,
* Programmable gathers and color transforms.

`XPHMG_GFX` does not define a fixed graphics pipeline.
Coordinate transforms, control flow, batching, and memory orchestration are expressed explicitly using RSV, MTX, and XMEM.

All numeric and memory behavior is governed by the effective state of `XPHMG_CAP`, ensuring consistent interpretation across graphics, compute, and geometry workloads.

---

## 2. Dependencies

**Mandatory:**

- `XPHMG_CAP` -- authoritative numeric and memory policy (`CAP.PREC.*`, `CAP.XMEM.*`, `CAP.XMEM.SVM*`)
- `XPHMG_RSV` -- Simple Vector windows and masks
- `XPHMG_MTX` -- matrix multiply/transform ops
- `XPHMG_XMEM` -- DMA/LDS/descriptor plumbing via CAP state

**Optional:** `XPHMG_RT` (shading path), descriptor-based offload frameworks built on `XPHMG_MEMCTL`.

`XPHMG_GFX` does not restate CAP fields; it consumes the effective CAP state latched at the instruction boundary.

---

## 3. Design Goals

- Minimal instruction footprint -- single ops perform multi-step work (fetch + filter + convert)
- Unified numeric control via `CAP.PREC.*` (MODE/ALT/STAT/EXC)
- CAP-driven memory defaults (coherence domain, classes, streaming profiles, descriptor policy, SVMEM)
- LMEM-first architecture; early culling using RSV predicate masks
- Predictable DMA/fence behavior aligned with XMEM
- Optional extensions for programmable filters, LUT color transforms, and hierarchical Z

---

## 4. GFX CSRs (vendor window)

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

> The numerical and memory behavior of all CSRs above is governed by CAP. Address placement must avoid the CAP window `0x7C0-0x7FF`; if that window is occupied by CAP on the target, remap GFX CSRs to a vendor-private block while preserving field semantics.

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

---

## 5. Instruction Set (major opcode **0x2B**, I-type)

All instructions use the 32-bit I-type encoding:  
`imm[11:0] | rs1 | funct3 | rd | 00101011b`.

Precision, element width, accumulator width, saturation, quantization, NaN policy, ZMODE, SAE, and trap enables are taken from `CAP.PREC.*` (effective state in `CAP.PREC.STAT`). `ppp` selects precision; `ppp=111` uses the active CAP effective state. Optional ALT selection follows `CAP.PREC.ALT`.

Z value format and width are governed by CAP effective precision. Compare/reverse-Z/early-late are configured in `GFXZCFG`.

**Precision selection.** When `ppp=111`, the instruction uses the effective state derived from `CAP.PREC.MODE/ALT` at decode. If the per-op `ALT` bit is present and set (or CAP ALT.EN is active), the alternate mapping in `CAP.PREC.ALT` applies. For encodings that expose per-op advisory precision (`ppp!=111`), the implementation must reconcile the request with CAP rules. If the requested PET/EW cannot be realized under CAP policy, the operation traps as Unsupported (see section 13) and sets `CAP.PREC.STAT.UNSUP_FMT`.

### Optional ALT selector bit (no opcode change)

One `imm` bit may be reserved per instruction as `ALT`:

- `ALT=0` -> use `CAP.PREC.MODE`
- `ALT=1` -> use `CAP.PREC.ALT` merged effective state

If omitted, bank choice follows `CAP.PREC.ALT` policy only.

---

### 5.1 `GFX.SAMPLE` -- Texture/tensor sampling

```
ppp m f w | xxxxx | 010 | xxxxx | 00101011
```

- `ppp`: precision selector (CAP effective state).
- `m`: filtering (0 = nearest, 1 = bilinear).
- `f`: color-space (0 = linear, 1 = sRGB->linear if enabled).
- `w`: wrapping override (0 = use CSR, 1 = use imm[3:2]).

**Semantics:**  
`rd <- SAMPLE(TEX, coords from rs1, base/pitch from CSRs or descriptor)`.

- Accepts either CSR-bound or pointer-to-descriptor mode.
- Coordinates follow normalized (`[0,1)`) or texel-space depending on `GFXCFG.NORM`.
- Fractional parts are used for bilinear filtering.
- Out-of-range obeys `GFXWRAP`; `border` returns `GFXBORDERC`.
- If `f=1` and `SRGB_EN=1`, applies sRGB->linear post-fetch.

---

### 5.2 `GFX.SAMPLE.GATH` -- Programmable gather pattern

```
ppp n p s | xxxxx | 010 | xxxxx | 00101011
```

- `ppp`: precision (CAP effective state).
- `n`: number of taps (from imm or descriptor).
- `p`: weighted accumulate (0=average, 1=use weights from table).
- `s`: signed offset mode.

**Semantics:**  
Gathers `n` texels according to offset table in `GFXGATHBASE` (each entry: `du, dv, weight`). Results accumulate in the effective accumulator width (CAP-driven) and are down-converted on writeback under CAP rules (SAT, NAN_POL, FP_RMODE, quantization).

**Offset table format (at `GFXGATHBASE`)**  
Each tap entry is `{du, dv, weight}` with PET/EW from CAP. For integer PET, `du/dv` use fixed-point per EW; for floating PET, they are FP. Maximum taps `n` is implementation-defined but must be >= 8; values above the implementation limit are truncated (setting `CAP.PREC.STAT.UNSUP_FMT` is allowed).

---

### 5.3 `GFX.INTERP` -- Bilinear/perspective interpolation

```
ppp M P B | xxxxx | 011 | xxxxx | 00101011
```

- `M`: 0 = bilinear, 1 = perspective.
- `P`: premultiplied-alpha variant.
- `B`: bias enable (adds bias post-interp).

**Semantics:**  
`rd <- interp2D(tex00, tex01, tex10, tex11, ufrac, vfrac [,bias])`.

Inputs and outputs follow CAP precision; accumulation uses CAP effective ACCW with CAP rounding/NAN_POL.

---

### 5.4 `GFX.SHADE` -- Arithmetic shading / lighting combine

```
ppp OO L N | xxxxx | 100 | xxxxx | 00101011
```

- `OO`: op (`00=add`, `01=sub`, `10=mul`, `11=dot`).
- `L`: lighting enable (uses normal/light RSV banks).
- `N`: normalize result.

**Semantics:**  
`rd <- shade(op, srcA, srcB [,normal/light])`, honoring CAP precision, rounding, saturation, and quantization settings.

---

### 5.5 `GFX.ZTEST` -- Depth comparison

```
ppp w e m | xxxxx | 101 | xxxxx | 00101011
```

- `w`: write enable (0=test only, 1=update).
- `e`: early-Z mode flag (test before shading).
- `m`: mask update policy (0=replace, 1=mask&=pass).

**Semantics:**  
Compares `z_in` from RSV or LMEM with `Zbuffer` (per `GFXZCFG.CMP`). Writes new Z if passed and `w=1`. Returns per-lane mask of survivors; optionally updates RSV predicate. Numeric format and NaN handling follow CAP effective precision and `CAP.PREC.MODE.NAN_POL`.

**Atomicity & ordering:**  
If `w=1`, `GFX.ZTEST` performs a compare-and-conditional-write that is atomic per pixel within the core. Visibility to LMEM/DRAM follows the fence scope (see `GFX.FENCE`). Cross-core atomicity is guaranteed only when the Z buffer resides in LMEM with single-writer ownership or when the fabric provides atomic writes for the configured width.

---

### 5.6 `GFX.ZWRITE` -- Write Z-buffer (explicit)

```
0000 00 01 00 | xxxxx | 101 | xxxxx | 00101011
```

Writes per-lane depth values (from RSV) to `Zbuffer`, respecting pitch and CAP precision. Does not recheck CMP; use `GFX.ZTEST` for guarded updates.

---

### 5.7 `GFX.HZ.BUILD` -- Hierarchical-Z reduce

```
ppp tt ll hh | xxxxx | 001 | xxxxx | 00101011
```

- `tt`: reduction type (`00=min`, `01=max`).
- `ll`: level of hierarchy to build/update.
- `hh`: tile height/width exponent (log2).

**Semantics:**  
Reduces Zbuffer tiles into a coarser hierarchy level in LMEM or DRAM, producing min/max depth per tile for early rejection. Internal accumulation may be promoted per CAP ACCW; outputs follow CAP PET/EW.

---

### 5.8 `GFX.LUT1D` / `GFX.LUT3D` -- Color LUT transform

```
ppp d i c | xxxxx | 100 | xxxxx | 00101011
```

- `d`: LUT dimension (0=1D, 1=3D).
- `i`: interpolation (0=nearest, 1=linear/trilinear).
- `c`: channel mode (0=RGB, 1=RGBA or custom).

**Semantics:**  
Applies color transform from LUT table in `GFXLUTBASE`/`GFXLUTFMT`. Uses normalized indices (0..1) or texel-space indices depending on CSR. Results are written back to RSV, respecting CAP precision, rounding, NaN, and saturation rules.

---

### 5.9 `GFX.FENCE` -- Pipeline barrier

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

---

### 5.10 Culling / Visibility Instructions

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

Combines predicate masks (`AND/OR/XOR/COPY`).

#### `GFX.CULL.COMPACT`

```
mm aa rr cc | xxxxx | 000 | xxxxx | 00101011
```

Stream compaction; writes only surviving lanes to LMEM (indices and/or attributes). If `cc=1`, returns count in `rd`. Masked-lane behavior follows CAP ZMODE.

**Clip W handling:** with `z=1`, lanes with `!isfinite(w)` or `w <= GFXCLIP_EPS` are rejected before plane tests.

---

## 6. Numeric & Memory Model (CAP-driven)

All GFX units (Sampler, Interp, Shade, Z, LUT, Gather) consume only the effective state from `CAP.PREC.*` and only the memory defaults from `CAP.XMEM.*` / `CAP.XMEM.SVM*`.

- **PET/EW/ACCW:** Derived from `CAP.PREC.MODE/ALT` at the instruction boundary.
- **Rounding (`FP_RMODE`), saturation (`SAT`/`ALT_SAT`), quantization (`Q`, zero-point, scale-shift), NaN policy (`NAN_POL`), ZMODE, SAE, and FP trap enables:** Follow CAP effective state (`CAP.PREC.STAT` and `CAP.PREC.EXC.EN`).
- **Accumulator policy:** If CAP effective ACCW indicates promotion (e.g., ACCW=FP32 or auto), units must promote internal accumulations accordingly. If fixed ACCW is selected and insufficient, the caller must choose an appropriate CAP mode.
- **Masked lanes:** Honor CAP ZMODE (merge vs zero) for all outputs, including sampler/gather/interp/shade/LUT and Z pipeline writes.
- **Stickies/traps:** FP/precision stickies are reported in `CAP.PREC.EXC.ST` and `CAP.PREC.STAT`; traps occur only if enabled and SAE is not set.
- **Memory semantics:** Coherence domain, memory class selection, streaming detection and profiles, LDS budgets, compression defaults, descriptor defaults, and SVMEM behavior all follow `CAP.XMEM.*` and `CAP.XMEM.SVM*`. GFX must not implement divergent memory policy.

**Processing order (canonical):**  
`fetch/convert -> filter/interp/gather -> color-space convert -> pack/saturate`

---

## 7. Sampling & Coordinate Conventions

- Coordinates normalized `[0,1)` when `GFXCFG.NORM=1`; otherwise texel-space.
- Texel centers at `i+0.5`.
- Bilinear uses `fract(u), fract(v)`.
- Out-of-range follows wrapping policy; `border` uses `GFXBORDERC`.
- sRGB handled when both `f=1` and `SRGB_EN=1`.
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
If `rs1` points to `TextureDesc`, it overrides all equivalent CSRs for that instruction. Descriptor memory accesses use CAP XMEM defaults (class, domain, streaming profile, compression, descriptor policy).

---

## 8. RSV Window Conventions

| Instruction  | Input Windows      | Output Windows | Notes                    |
| ------------ | ------------------ | -------------- | ------------------------ |
| `GFX.SAMPLE` | `RSV a0..a1` (u,v) | `RSV r0..r3`   | RGBA sample              |
| `GFX.INTERP` | `r0..r3`, `a2..a3` | `r4..r7`       | Bilinear/perspective     |
| `GFX.SHADE`  | `r*`, `n*`, `l*`   | `r*`           | Lighting                 |
| `GFX.ZTEST`  | `rZ`               | mask           | May update RSV predicate |
| `GFX.LUT*`   | `rRGB`             | `rRGB`         | In-place color transform |

ZMODE from CAP applies to masked lanes for all operations.

---

## 9. Execution Model (Informative)

### 9.1 Typical Flow (2D/3D)

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

### 9.2 Wavefront Queues & Hierarchical-Z (Informative)

- **Wavefront queues:** Implemented as LMEM ring buffers. Push survivors via `GFX.CULL.COMPACT`. Memory class, streaming profile, and compression obey CAP XMEM defaults or per-descriptor overrides.
- **Hierarchical Z:** `GFX.HZ.BUILD` reduces tiles asynchronously for future early-Z passes. Tile fetch/store uses CAP XMEM settings.

---

## 10. Peer LMEM Bus Hints (Informative)

`GFXPEERHINT` provides optional guidance to inter-tile or multi-core fabrics:

| Field          | Meaning                             |
| -------------- | ----------------------------------- |
| `peer_id`      | Target peer LMEM ID.                |
| `qos`          | Quality-of-service / priority hint. |
| `prefetch`     | Suggest prefetch size or stride.    |
| `non_temporal` | Disable caching if set.             |

Hints are advisory only; correctness must not depend on them.

---

## 11. Cross-Domain Interoperability & Implementation Requirements (Normative)

This section aligns GFX with CAP/RSV/MTX/RT rules. All items are mandatory unless stated otherwise.

### 11.1 State Latching at Instruction Boundary

Each GFX instruction must snapshot at decode:

- `CAP.PREC.MODE`
- `CAP.PREC.ALT`
- `CAP.PREC.EXC.EN`
- Relevant `CAP.XMEM.*` and `CAP.XMEM.SVM*` fields (coherence domain, memory classes, streaming profiles, LDS budgets, compression defaults, descriptor defaults, SVMEM parameters)

Mid-instruction CSR changes must not affect the in-flight operation.

### 11.2 Unified Numeric Semantics (CAP.PREC)

GFX must interpret PET/EW/ACCW, rounding, saturation, quantization, NaN policy, ZMODE, SAE, and trap enables exactly as defined by CAP effective state (`CAP.PREC.STAT`). No GFX-specific numeric variants are allowed. Internal promotion to FP32 for quality is allowed, but outputs must conform to CAP format and policy, updating stickies accordingly.

### 11.3 Mask & Predication (ZMODE)

Masked lanes follow CAP ZMODE:

- `ZMODE=0` -> masked-off elements preserve destination (merge)
- `ZMODE=1` -> masked-off elements write architectural zero

Applies to sampler, gather, interp, shade, LUT, Z pipeline, and culling/compaction results.

### 11.4 FP Exceptions, NaN, and Trap Policy

GFX must update `CAP.PREC.EXC.ST` and `CAP.PREC.STAT` for `NV,DZ,OF,UF,NX,SNAN_SEEN,QNAN_SEEN,SAT_HIT,FTZ_HIT,DOWNCAST_TAKEN,UNSUP_FMT` using the same rules as RSV/MTX/RT. Traps occur only if enabled in `CAP.PREC.EXC.EN` and effective SAE is 0; otherwise stickies are set. NaN propagation/squash follows `CAP.PREC.MODE.NAN_POL`.

### 11.5 Memory Semantics via CAP.XMEM

All GFX memory accesses (texture fetch, Z/LUT/gather loads and stores, descriptor reads, compaction writes) must:

- Obey `CAP.XMEM.DOM` coherence domain
- Use memory class mapping from `CAP.XMEM.CLSMAP` and default class/QoS from `CAP.XMEM.DESCPOL`
- Apply streaming detector and profiles (`CAP.XMEM.STREAM_CTL`, `CAP.XMEM.SPROF*`)
- Respect LDS budgets (`CAP.XMEM.LDS_BUDGET`) and compression defaults (`CAP.XMEM.COMP_CTL`)
- Use SVMEM rules (`CAP.XMEM.SVM*`) for any indexed gather/scatter path, including masked-lane ZMODE and FOF behavior

No GFX-local overrides are permitted unless provided by a descriptor field that explicitly replaces the CAP default.

### 11.6 Descriptor Overrides

Descriptors may override only the fields they contain (e.g., base/pitch/format/wrap, optional memctl block). Unspecified fields fall back to CAP defaults. If a descriptor includes an `XPHMG_MEMCTL` trailer, only `dom`, `mclass`, `sprof_id`, `hyb`, `sec` override CAP; other memory semantics remain bound to CAP.

### 11.7 GFX-Specific Numeric Requirements

- Sampler, gather, and LUT units may use FP32 internal accumulation when CAP ACCW or PET/EW would otherwise lose precision; outputs must still follow CAP policies and stickies.
- Z pipeline writes must downcast per CAP PET/EW and honor CAP saturation and NaN policy.
- Color-space conversion must not bypass CAP NaN/FTZ/quantization rules.
- Advisory precision bits in encodings must never change CAP state; they only request a mapping. If unsupported, trap per section 13.

### 11.8 Apply Semantics & Determinism

- `CAP.PREC.MODE/APPLY0` and `CAP.PREC.ALT/APPLY1` take effect atomically; GFX must not observe partial writes.
- XMEM/SVMEM CSRs become effective at the next instruction boundary; GFX must latch them once per instruction.
- Debug/replay must see a stable effective state (mirror of `CAP.PREC.STAT` and current `CAP.XMEM.*`).

---

## 12. Encodings (Summary)

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
| `GFX.CULL.*`        | (see section 5.10) | varies | `00101011` | Culling/compaction ops  |

---

## 13. Exceptions and Status

| Condition                                                                          | Action                                                                                         | Source of control |
| ---------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ----------------- |
| Unsupported combination of PET/EW/ACC requested by instruction or CAP effective    | **Illegal Instruction** (or **Unsupported Feature**) and set `CAP.PREC.STAT.UNSUP_FMT=1`       | CAP               |
| Overflow/underflow/inexact/NaN per CAP effective precision                         | Update `CAP.PREC.EXC.ST` stickies (trap only if enabled in `CAP.PREC.EXC.EN` and SAE=0)        | CAP               |
| Masked-lane zero/merge mismatch                                                    | Behave per CAP ZMODE; any divergence is an architectural bug                                   | CAP               |
| Invalid LMEM/DRAM access                                                           | **Load/Store Access Fault**                                                                    | Memory subsystem  |
| GFX disabled (`GFXCFG.EN=0`)                                                       | No-op (or optional trap)                                                                       | GFXCFG            |

Sticky precision faults are visible in `CAP.PREC.STAT` when implemented.

---

*This specification is part of the PHMG Open Specification Set (CC-BY-SA 4.0).* 
