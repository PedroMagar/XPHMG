# RISC-V Vendor Extension **XPHMG_MTX**
**Matrix/Tensor Core for Small M×N Transforms and Inference Primitives**

**Category:** Vendor Extension (`XPHMG_MTX`)  
**Version:** 0.1.0
**Author:** Pedro H. M. Garcia  
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview (Informative)

`XPHMG_MTX` defines a compact **matrix and tile compute execution path** for small M×N transforms and dense arithmetic kernels.

The extension targets workloads common in graphics, vision, and machine learning inference, where small tiles and high data reuse dominate.

MTX instructions perform computation only.
All data movement, residency, and streaming behavior is handled explicitly via `XPHMG_XMEM`, and all numeric behavior is governed by `XPHMG_CAP`.

Rather than encoding fixed shapes into the opcode space, MTX expresses tile dimensions and composite shapes explicitly, enabling a uniform execution model across small matrices and tiled compute workloads.

MTX is designed to interoperate naturally with RSV-controlled loops, RT geometry transforms, and GFX shading pipelines.

---

## 2. Dependencies

- **Required:** `XPHMG_CAP` precision block (`CAP.PREC.MODE`, `CAP.PREC.ALT`, `CAP.PREC.STAT`)  
- **Recommended:** `XPHMG_XMEM` (LDS dynamic, class partition, streaming profiles)  
- **Optional:** `XPHMG_RT` (for hybrid RTU+NPU pipelines), `RSV` (vector loop control)

---

## 3. Design Principles

- **Small-core, high-reuse:** single 4×4 MAC datapath time-multiplexed for smaller/rectangular shapes.  
- **Precision-neutral ISA:** all numeric behavior is owned by `XPHMG_PREC`; opcodes carry no format bits unless explicitly stated.  
- **Descriptor-first I/O:** data motion is handled by XMEM (`XMEM.LDG/STG/PREF`, `XLDS.*`, `XSWZ.*`) or by host code; MTX instructions only compute.  
- **Low-latency per tile:** suitable for per-vertex transforms, per-pixel blocks, and small tensor tiles.

---

## 4. Matrix Configuration CSRs (0x7D8–0x7DF)

| Address  | Name      | Access | Description                                                                                                           |
|----------|-----------|--------|-----------------------------------------------------------------------------------------------------------------------|
| 0x7D8    | `MTXCFG`  | RW     | Global flags: `TRANSPOSE_DEF(0)`, `BIAS_EN(1)`, `SAT_EN(2)` (hint; effective saturation is governed by `XPHMG_PREC`). |
| 0x7D9    | `MTXSEL`  | RW     | Active matrix bank selector (0–3).                                                                                    |
| 0x7DA    | `MTX2`    | RW     | Banked storage for 2×2 (4 elements, packed by `XPHMG_PREC` formats).                                                  |
| 0x7DB    | `MTX3`    | RW     | Banked storage for 3×3 (9 elements; rectangular shapes use sub-blocks).                                               |
| 0x7DC    | `MTX4`    | RW     | Banked storage for 4×4 (16 elements).                                                                                 |
| 0x7DD    | `MTXBIAS` | RW     | Optional bias vector (2–4 elements, packed).                                                                          |
| 0x7DE    | `MTXSTAT` | RO     | Status/counters: `ops_issued`, `ops_saturated`, `ftz_events`, `last_shape`.                                           |
| 0x7DF    | `MTXCAP`  | RO     | Capabilities: `HAS_MUL`, `HAS_MAD`, `HAS_BIAS`, `MAX_SHAPE(2b)` (2×2..4×4).                                           |

> **Packing & endianness**: element packing/size follows the **effective precision** (`XPREC_STAT`). SW may load via normal stores or use `MTX.LD` convenience instruction.

---

## 5. Precision Model Integration

All numeric behavior comes from **XPHMG_PREC**:

- Program **base** and **alternate** formats via `XPREC_MODE` / `XPREC_ALT`, then verify with `XPREC_STAT`.  
- Fields such as `ACCW` (accumulator width), `SAT`, `FP_RMODE`, `DENORM_FTZ`, `Q`, `PACK` are **not redefined** here.  
- Implementations **shall** execute MTX ops using the **effective** precision reported by `XPREC_STAT`.

---

## 6. Data Movement & Memory (with XPHMG_XMEM)

- Use `XLDS.ALLOC/FREE/BAR/CPY` to stage tiles into **LDS**.  
- Use `XMEM.PREF/LDG/STG` (and `XMEM.DLOADC/DSTOREC` if enabled) to manage cache/streaming/compression.  
- For hybrid MTX↔RT passes, set `XPHMG_MEMCTL.{mclass, sprof_id, hyb, dom}` in **descriptors** to steer partitioning/QoS (CL0..CL3).

> MTX instructions **do not** perform DMA. This is by design.

### 6.1 Tile Shape Immediates

All tile-level MTX helper operations (transpose, epilogue, reductions) operate on a logical M×N tile expressed in terms of the canonical XDL 4×4 base tiles.

Rather than encoding the tile shape in the opcode name (e.g., `…4x4`, `…8x8`), the tile shape is carried by small immediates attached to the instruction.

Conceptually, each helper instruction carries two small unsigned fields:

- `m_blk` — number of 4-row blocks (step of 4 in the row dimension),
- `n_blk` — number of 4-column blocks (step of 4 in the column dimension).

The effective tile dimensions are:

- `M = 4 × m_blk` rows,
- `N = 4 × n_blk` columns,

and the tile is understood as a composite XDL tile (`XDL_TILE(M,N)`) built from `m_blk × n_blk` canonical 4×4 sub-tiles in row-major tile order (see XDL spec).

> **Note:** Typical encodings will use 3-bit fields for `m_blk` and `n_blk` (values 1–8), allowing logical shapes up to 32×32. The exact encoding of these immediates is implementation-defined, as long as:
>
> * `m_blk ≥ 1`, `n_blk ≥ 1`,
> * `M` and `N` are multiples of 4,
> * and `(M, N)` do not exceed the maxima advertised by `CAP.TILE` (`MAX_M_4X`, `MAX_N_4X`).

Constraints:

- If `COMPOSITE = 0` in `CAP.TILE`, only `M = N = 4` is architecturally valid for helper operations (`m_blk = n_blk = 1`); other shapes MUST raise an *illegal instruction* trap.
- If `COMPOSITE = 1`, any `(M,N)` with:
  - `M = 4 × m_blk`, `N = 4 × n_blk`,
  - `M ≤ 4 × (MAX_M_4X + 1)`, `N ≤ 4 × (MAX_N_4X + 1)`,
  
  is architecturally valid; the implementation MAY decompose the helper into multiple 4×4 sub-operations internally.


### 6.2 Tile Transpose Helper (`MTX_TRANSPOSE`)

`MTX_TRANSPOSE` is a generic tile-level helper that transposes an M×N composite tile, where both dimensions are multiples of 4 and are encoded by the tile-shape immediates (`m_blk`, `n_blk`).

#### 6.2.1 Scope

The instruction takes:

- A source tile `T_src` laid out in an `XDL_TILE(M,N)` layout (composed of `m_blk × n_blk` 4×4 sub-tiles in row-major tile order),
- A destination tile `T_dst` with dimensions `N×M` in the corresponding `XDL_TILE(N,M)` layout,
- Tile-shape immediates `m_blk`, `n_blk` as defined in §6.1.

Effective dimensions:

- `M = 4 × m_blk` rows,
- `N = 4 × n_blk` columns.

The special case `m_blk = n_blk = 1` corresponds to a single 4×4 tile.

#### 6.2.2 Semantics

For all `0 ≤ i < M` and `0 ≤ j < N`:

```

T_dst[j][i] = T_src[i][j]

```

where indices `(i,j)` and `(j,i)` are interpreted in the logical M×N / N×M matrix space, and both `T_src` and `T_dst` are stored using the canonical XDL composite-tile layout (see XDL spec).

Implementations are free to decompose the transpose into:

- 4×4 in-tile transposes, plus
- permutations of 4×4 sub-tiles,

as long as the final visible contents of `T_dst` match the mathematical transpose of the M×N matrix.

#### 6.2.3 Valid Shapes

- If `COMPOSITE = 0` in `CAP.TILE`, only `m_blk = n_blk = 1` (4×4) is valid.
- If `COMPOSITE = 1`, any `(M,N)` allowed by `CAP.TILE` (`MAX_M_4X`, `MAX_N_4X`) is valid. Larger shapes MUST trap as illegal.

Tiles involved in `MTX_TRANSPOSE` MUST reside in XMEM regions whose layout metadata indicates an XDL tile layout (`XDL_TILE(M,N)` / `XDL_TILE(N,M)`); the instruction MUST NOT be applied to non-tile layouts.

### 6.3 Epilogue Stage (Bias / Clamp / Activation)

Epilogue operations act on an M×N tile, where M and N are determined by the tile-shape immediates (`m_blk`, `n_blk`) as described in §6.1:

- `M = 4 × m_blk`, `N = 4 × n_blk`,
- The tile is stored in `XDL_TILE(M,N)` layout in XMEM.

An epilogue instruction reads `T` (the just-computed tile) and optionally:

- A bias term (scalar, per-row or per-column), and
- Clamp thresholds (e.g. MIN/MAX, often 0 / +∞ or 0 / 1),

and produces a new tile `T'` of the same M×N shape and layout.

Minimal recommended semantics:

1. **Bias-add**  
   For each `0 ≤ i < M`, `0 ≤ j < N`:

```

T'[i][j] = T[i][j] + bias(i,j)

```

where `bias(i,j)` is computed according to the selected broadcast mode (scalar / row / column / per-element).

2. **Clamp**  

```

T''[i][j] = clamp(T'[i][j], MIN, MAX)

```

with MIN/MAX either fixed or configured via MTX configuration CSRs.

All arithmetic obeys the current CAP precision policy (PET/EW/ACCW, FTZ, SAT, FP_RMODE, etc.).

Implementations MAY fuse the epilogue with MAC operations or execute it as a separate instruction. Architecturally, the visible result must be equivalent to applying bias-add followed by clamp on the M×N tile.

### 6.4 Tile Reductions

Reduction helpers operate on an M×N tile, with dimensions defined by `m_blk`, `n_blk` (§6.1):

- `M = 4 × m_blk`, `N = 4 × n_blk`,
- Tile stored in `XDL_TILE(M,N)` layout.

Minimal recommended reductions:

1. **Full-tile SUM**

```

s = Σ_{i=0..M-1} Σ_{j=0..N-1} T[i][j]

```

where `s` is a scalar result written to a destination register or a 1×1 tile.

2. **Row-wise SUM**

For each row `i`:

```

r[i] = Σ_{j=0..N-1} T[i][j]

```

producing an M-element vector (which may itself be stored as a 1×M or M×1 XDL tile in XMEM).

Implementations MAY extend reductions to columns and other operators (MAX/MIN, etc.) but these are optional.

Numeric behavior (rounding, FTZ, saturation, NaN policy) MUST follow the active CAP precision policy. If saturation or other exceptional events occur, the corresponding CAP.PREC sticky flags MUST be updated accordingly.

As with transpose and epilogue, if `COMPOSITE = 0` only 4×4 reductions are valid; otherwise, any M×N within `CAP.TILE` maxima is valid.

---

## 7. Instruction Set (major opcode `CUSTOM-2` — suggested `0x5B`)

All encodings are I-type 32-bit unless noted. Final `funct` values remain in the vendor table.

### 7.1 `MTX.LD` — Load Matrix Bank from Memory (convenience)

```

imm[11:0] | rs1 | funct3=000 | rd=x0 | opcode=01011011

```

**Fields (in `imm`):**
- `SHAPE[3:0]`  (2×2=0x22, 2×3=0x23, 3×2=0x32, 3×3=0x33, 2×4=0x24, 4×2=0x42, 4×4=0x44)  
- `TRANSPOSE`   (bit 4)  
- `ROWMAJOR`    (bit 5)  
- `WITH_BIAS`   (bit 6)

**Semantics:** Loads matrix elements (and optional bias) from `[rs1]` into the **active bank** (`MTXSEL`). Element interpretation uses the **effective** precision (`XPREC_STAT`). Rectangular shapes map to sub-blocks of `MTX3/MTX4`.

> SW may also fill `MTXx` via ordinary `csrw` stores; `MTX.LD` is a helper.

---

### 7.2 `MTX.MUL` — Matrix × Vector Multiply

```

imm[11:0] | rs1 | funct3=001 | rd | opcode=01011011

```

**Fields (in `imm`):**
- `SHAPE[3:0]` (same codes as above)
- `TRANSPOSE`  (bit 4) — XOR with `MTXCFG.TRANSPOSE_DEF`
- `SAT_HINT`   (bit 5) — hint; final saturation from `XPHMG_PREC`
- `VEC_COUNT[2:0]` (bits 8:6) — number of contiguous vectors to process (1..8)
- `STRIDE_BYTES[1:0]` (bits 10:9) — 0=packed, 1=elem, 2=row, 3=impl.

**Operands:**
- `rs1` = base address of input vector(s)  
- `rd`  = base address for output vector(s)

**Semantics:**  
Compute `u = M × v` for `VEC_COUNT` vectors, using the **active bank** and the **effective** precision. Stride hints help HW prefetch; correctness does not depend on them.

---

### 7.3 `MTX.MAD` — Matrix × Vector + Bias (optional)

```

imm[11:0] | rs1 | funct3=100 | rd | opcode=01011011

```

Same as `MTX.MUL`, with an added bias from `MTXBIAS` bank: `u = (M × v) + bias`.  
Presence is indicated by `MTXCAP.HAS_MAD`.

### 7.4 Canonical Tile Output Layout

All MTX instructions that write tile results into XMEM MUST produce data in the canonical XDL tile layout:

- **4×4 row-major tiles** (`XDL_TILE4x4_ROWMAJOR`), or  
- Valid composite-tile arrangements as defined in the XDL model (`XDL_TILE(M,N)` composed of multiple 4×4 tiles).

This layout is REQUIRED for Tier-0 cross-engine handoff.  
MTX MUST NOT use proprietary or implementation-private tile encodings when writing into Tier-0.

#### Linking Behavior

When an MTX operation targets a composite tile, implementations MAY:

1. Compute only the required 4×4 tiles, OR  
2. Fuse multiple tiles into a single wide operation, if hardware supports it.

In all cases, the architectural result MUST match the composite-tile definition.

---

## **8. Cross-Domain Interoperability & Implementation Requirements (Normative)**

### **8.1 Precision / Numeric State (CAP → MTX)**

Every MTX instruction (`MTX.LD`, `MTX.MUL`, `MTX.MAD`, and vendor-extended variants) **must snapshot the full effective precision state** at instruction boundary:

* `CAP.PREC.MODE`
* `CAP.PREC.ALT`
* `CAP.PREC.STAT`
* `CAP.PREC.EXC.{EN,ST}`

From these, MTX derives:

* **Effective PET/EW** (element type/width),
* **Effective ACCW** (accumulator width),
* **Rounding mode** (`EFF_FP_RMODE`),
* **Quantization state** (`EFF_Q`, ZP, SCALE_SH),
* **NaN policy**,
* **ZMODE** (zero/merge) for masked writeback,
* **SAE behavior** (trap vs no-trap),
* **UNS** for integer signedness.

#### **Conformance rules:**

1. MTX must not reinterpret FP or INT formats in ways not defined by CAP.
2. Mixed-precision GEMMs (FP8→FP16→FP32, FP16→FP32, INT8→INT32) must follow **ALT mode & MIXED** rules from CAP.
3. Widening accumulation must respect `EFF_ACCW`.
4. Down-casts must set `DOWNCAST_TAKEN` or `UNSUP_FMT` if outside supported domain.

#### **Forbidden behaviors:**

* Custom MTX rounding rules,
* MTX-local NaN propagation rules,
* MTX-local quantization,
* Independent saturation/overflow behavior.

MTX **only** uses CAP’s effective precision semantics.

### **8.2 Mask & ZMODE Semantics**

When MTX uses masking (for partial-tile compute, edge tiles, or RSV-driven launches):

* Masked elements must follow `CAP.PREC.MODE.ZMODE`:

| ZMODE           | Behavior                                          |
| --------------- | ------------------------------------------------- |
| **0 = merge**   | masked elements preserve prior accumulator value  |
| **1 = zeroing** | masked elements write architectural zero (per EW) |

This applies to:

* MTX.LD (masked tile loads),
* MTX.MUL / MTX.MAD partial-tile compute,
* Any MTX op using RSV-provided predicate masks.

MTX may **not** invent its own masked-lane semantics.

### **8.3 FP Exceptions & Sticky Flags**

MTX operations must report:

* `SAT_HIT`
* `FTZ_HIT`
* `DOWNCAST_TAKEN`
* `UNSUP_FMT`
* FP exception stickies: `NV`, `DZ`, `OF`, `UF`, `NX`, `QNAN_SEEN`, `SNAN_SEEN`

#### **Trap rules:**

* MTX must deliver traps exactly as CAP defines:

  * Trap only if `EXC.EN[bit]=1` **and** `EFF_SAE=0`.
* If SAE override applies (direct CAP or RSV prefix), MTX must **suppress trap** but still set sticky bits.

#### **First-fault semantics:**

MTX does **not** define FOF for math pipelines, but:

* If MTX uses RSV for tile indexing, the **RSV.SVFAULTI** rules apply.
* For MTX memory access FOF, see §7.4.

### **8.4 Unified Memory Behavior (CAP.XMEM → MTX)**

All MTX memory operations (`MTX.LD`, streaming tile loads, tiled scatter/stores, LDS-resident tiles) must obey:

* `CAP.XMEM.DOM`
* `CAP.XMEM.CLSMAP`
* `CAP.XMEM.LDS_BUDGET`
* `CAP.XMEM.STREAM_CTL`
* `CAP.XMEM.SPROF0..3`
* `CAP.XMEM.COMP_CTL`
* `CAP.XMEM.DESCPOL`
* All SVMEM gather/scatter controls

#### **Mandatory rules:**

1. **Coherence:** MTX tile loads use the coherence domain defined by `CAP.XMEM.DOM` unless explicitly overridden by descriptor.
2. **Streaming:** MTX hardware prefetch must use the same stride/prefetch rules as RSV/GFX.
3. **SVMEM:** If MTX loads tiles via indexed addressing, it must use **exactly the SVMEM mechanism**, including:

   * Predication,
   * Address scaling,
   * FOF behavior,
   * ZMODE for masked destinations.
4. **Compression:** If inline compression is enabled in `CAP.XMEM.COMP_CTL`, MTX must adopt the same default.
5. **Descriptors:** If MTX uses a descriptor tail (`XPHMG_MEMCTL`), only explicitly set fields override CAP defaults.

#### **FOF behavior for MTX memory ops:**

If an MTX tile load uses SVMEM and FOF=1:

* On the first faulting lane (tile element),

  * The index must be written to `SVFAULTI`,
  * The operation must abort,
  * Software may retry.

MTX must not implement a different FOF mechanism.

### **8.5 Cross-Domain Dataflow Guarantees (MTX ↔ RSV/RT/GFX/NPU)**

#### **MTX ↔ RSV**

* RSV may prepare tiles (via permutes, interleaves) that MTX consumes without reinterpretation.
* MTX outputs must be read by RSV with identical numeric rules.

#### **MTX ↔ RT**

* MTX transforms must not change NaN/FP behavior observable by ray tests.
* MTX outputs used for instancing/transforms in RT must preserve CAP precision.

#### **MTX ↔ GFX**

* GFX shading stages may use MTX for small MLP layers, skinning, or 3×3/4×4 transforms.
* Masking, ZMODE and FP behavior must remain consistent.

#### **MTX ↔ NPU**

* MTX small tiles must be numerically equivalent to the NPU pipeline using the same CAP precision.
* Software may fuse MTX and NPU GEMM paths without precision mismatch.

### **8.6 MTX Tile Layout, Addressing & LDS Rules**

To ensure all domains can share data:

#### **Tile layout rules (normative)**

1. Tile element format = **CAP effective PET/EW**.
2. Tile accumulator format = **CAP effective ACCW**.
3. Mixed-precision tiles must map to CAP.ALT rules (FP8→FP16→FP32, etc.).
4. A tile stored to memory must be readable by RSV/RT/GFX without reinterpretation.

#### **LDS allocation rules**

1. MTX tile banks must allocate via `XLDS.ALLOC` using the same class budget rules as RSV/RT/GFX.
2. Tile bank swizzle (if implemented) must honor `CAP.XMEM.CAP.BIDIR_SWZ`.
3. LDS barriers must use the same fence ordering:

   * Tile written → must issue `XLDS.BAR ≥CLUSTER` before visible to other domains.

### **8.7 Descriptor Integration (`XPHMG_MEMCTL`)**

If MTX is invoked through a descriptor:

* Only explicitly set fields override CAP:

  * `dom`
  * `mclass`
  * `sprof_id`
  * `hyb`
  * `sec`
* All unspecified fields revert to CAP.XMEM default behavior.

Descriptors must **not** introduce MTX-specific semantics.

### **8.8 Forbidden Divergences**

MTX must not:

* Implement alternative gather/scatter semantics,
* Change ZMODE behavior,
* Use private FP rounding/saturation rules,
* Use private coherence or streaming rules,
* Treat NaNs differently from CAP,
* Change interpretation of PET/EW/ACCW,
* Ignore SVMEM predication or FOF.

Any violation is a **non-conformant implementation**.

### **8.9 Debug & Deterministic Replay Requirements**

MTX must expose to debug:

* Latched CAP.PREC effective state,
* Latched CAP.XMEM effective state,
* MTXCFG/MTXSTAT registers,
* Tile bank contents (if architecturally visible).

Replay tools must see:

* Stable precision & memory state per decoded instruction,
* No partial application of CAP or XMEM state.

### **8.10 Summary**

The MTX engine must behave as **one more consumer** of the unified XPHMG architectural state:

* **Precision semantics from CAP**
* **Memory semantics from CAP.XMEM**
* **Mask/FOF/addressing via RSV’s SVMEM rules**

This ensures that MTX does not become a "separate subsystem" but a tightly integrated element of the heterogeneous XPHMG execution model.

---

## 9. Shapes and Banking

| Shape | Elements | Bank                |
|-------|----------|---------------------|
| 2×2   | 4        | MTX2                |
| 2×3   | 6        | MTX3 (top-left)     |
| 3×2   | 6        | MTX3 (top-left)     |
| 3×3   | 9        | MTX3                |
| 2×4   | 8        | MTX4 (first 2 rows) |
| 4×2   | 8        | MTX4 (first 2 cols) |
| 4×4   | 16       | MTX4                |

All packing (FP8/BF16/FP16/FP32, INT8/INT4) follows `XPHMG_PREC`.

---

## 10. Scheduler & Hybrid Interop (informative)

Typical **hybrid** pipeline:

1) RTU builds G-Buffer (class CL0),  
2) MTX denoiser or MLP pass (class CL1),  
3) RTU composite (CL0 or HYB CL2).

Drivers should set `memctl.mclass={0,1,2}`, `memctl.sprof_id` (BVH vs weights profile), and `memctl.hyb` for shared phases. Barriers: `XFENCE.SIGNAL/WAIT` + `XLDS.BAR ≥ CLUSTER` for LDS→L1/L2 visibility.

---

## 11. Programming Examples

### 11.1 2×3 affine (FP16 base, ACCW=FP32)

```asm
# Precision (XPHMG_PREC)
li  t0, ( (0b10<<22) /*ACCW=FP32*/ | (0b011<<19) /*PET=FP16*/ | (0b01<<17) /*EW=16*/ | (1<<31) )
csrw 0x7E0, t0   # XPREC_MODE.APPLY0=1

# Load matrix bank 0 from memory (row-major 2×3)
csrw 0x7D9, x0   # MTXSEL=0
li   a0, mtx23_ptr
MTX.LD  imm=(0x23 | (1<<5)), rs1=a0

# Multiply 64 vectors, packed stride
li   a1, vin_ptr
li   a2, vout_ptr
MTX.MUL imm=(0x23 | (0<<4) | (0<<5) | (0b111<<6) | (0<<9)), rs1=a1, rd=a2
```

### 11.2 FP8 E4M3 mixed (mul FP8, acc FP16), 4×4

```asm
# Alternate precision
li  t1, (1<<30) /*ALT_EN*/ | (0b0010<<26) /*FP8_E4M3*/ | (0b01<<24) /*ALT_ACCW=FP16*/ | (1<<22) /*MIXED*/ | (1<<31)
csrw 0x7E1, t1   # XPREC_ALT.APPLY1=1

# 4×4 transform of 8 vectors (contiguous)
MTX.MUL imm=(0x44 | (0<<4) | (0<<5) | (0b111<<6) | (0<<9)), rs1=a1, rd=a2
```

### 11.3 INT8×INT8 with zero-point (Q path enabled)

```asm
# Base mode: quant enabled
li  t0, (1<<25) | (1<<31)    # Q=1, APPLY0
csrw 0x7E0, t0
# Alt: INT4 or leave INT8 as base (example leaves INT8)
MTX.MUL imm=(0x33), rs1=a1, rd=a2
```

---

## 12. Exceptions & Status

* **Illegal Instruction**: unsupported shape when `MTXCAP.MAX_SHAPE` is smaller; `HAS_MAD=0` with `MTX.MAD`.
* **Precision downgrade**: reflected in `XPREC_STAT` flags (e.g., `UNSUP_FMT`, `DOWNCAST_TAKEN`, `SAT_HIT`, `FTZ_HIT`).
* **MTXSTAT** may increment `ops_saturated`/`ftz_events` for profiling.

---

## 13. Hardware Notes

* The 4×4 datapath may implement **FMA** for FP modes and **MAC** for INT modes.
* ACCW selects wider accumulators (e.g., FP32) independent of input EW.
* Rectangular shapes gate unused lanes; bias add (if present) happens in the accumulator stage.
* Back-to-back issues are allowed if LDS/cache provide operands (XMEM handles stalls/prefetch).

---

## 14. Encodings (summary)

| Instruction | I-Type (32b) | Description |            |       |       |                        |
| ----------- | ------------ | ----------- | ---------- | ----- | ----- | ---------------------- |
| `MTX.LD`    | `imm         | rs1         | funct3=000 | rd=x0 | 0x5B` | Load matrix bank       |
| `MTX.MUL`   | `imm         | rs1         | funct3=001 | rd    | 0x5B` | Matrix × vector        |
| `MTX.MAD`   | `imm         | rs1         | funct3=100 | rd    | 0x5B` | Matrix × vector + bias |

> Exact `imm` bit allocation is defined in §7.1/7.2.

---

## 15. Compatibility

- Precision control for MTX is provided through `XPHMG_PREC` and the associated CAP precision state.
- MTX data loads/stores are defined through standard memory operations and `XPHMG_XMEM` + LDS; bespoke `XREG` paths and legacy `XMEM.DMA`/`XMEM.LDGPR/STGPR` opcodes are outside this specification.

---
*CC-BY-SA 4.0 -- Open Specification maintained by Pedro H. M. Garcia.*
Designed to integrate with `XPHMG_PREC`, `XPHMG_XMEM`, `XPHMG_RT`, and optional `RSV` or `V`.
