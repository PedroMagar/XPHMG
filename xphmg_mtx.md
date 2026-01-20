# RISC-V Vendor Extension **XPHMG_MTX**
**Matrix/Tensor Core for Small MxN Transforms and Inference Primitives**

**Category:** Vendor Extension (`XPHMG_MTX`)  
**Version:** 0.1.1
**Author:** Pedro H. M. Garcia
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview (Informative)

`XPHMG_MTX` defines a compact matrix and tile compute execution path for small MxN transforms and dense arithmetic kernels. It targets graphics, vision, and machine learning inference workloads with small tiles and high data reuse. MTX instructions perform computation only; all data movement, residency, and streaming behavior are handled by `XPHMG_XMEM`, and numeric behavior is governed by `XPHMG_CAP`. Tile dimensions and composite shapes are carried by immediates rather than baked into opcode variants. MTX interoperates with RSV-controlled loops, RT geometry transforms, and GFX shading pipelines without reformatting.

### 1.1 Conceptual Model

* MTX treats matrices and tiles as logical MxN shapes built from canonical 4x4 subtiles. Shape is carried by immediates rather than opcode variants.
* Compute-only: MTX opcodes do not allocate, move, or stream data; XMEM/RSV descriptors govern residency and access.
* Numeric behavior is precision-neutral: opcodes carry no format fields unless stated; element and accumulator formats come from CAP effective state.
* MTX reuses unified architectural state (CAP, XMEM, PMASK predicates) rather than introducing profile-local state.

### 1.2 Architectural Guarantees & Invariants

* MTX must not override CAP numeric policy or XMEM memory policy. Effective precision, rounding, saturation, NaN/FTZ, quantization, and ZMODE come from CAP at the instruction boundary.
* Tile shapes and bank mappings must honor `MTXCAP.MAX_SHAPE` and `CAP.TILE` limits; illegal shapes trap.
* Predication consumes PMASK effective predicates; masked lanes follow CAP.PREC.MODE.ZMODE and XMEM predication rules and must not use alternate semantics.
* MTX introduces no architectural CSRs beyond the MTX bank/config registers in Section 4.

### 1.3 Interaction with CAP / XMEM / Other Extensions

* CAP supplies PET/EW/ACCW, rounding, exception enables/stickies, and ZMODE; MTX instructions consume these without reinterpretation.
* XMEM controls residency, coherence domains, streaming profiles, and LDS usage for tiles; MTX relies on XMEM/descriptor state for data motion.
* RSV may drive indexing and gather/scatter for MTX tiles; predication consumes PMASK effective predicates (pbank selected by prefix or instruction field); results are consumable by RSV/RT/GFX/ACCEL without reformatting when stored in canonical layouts.

### 1.4 Undefined / Reserved Behavior

* Shapes beyond `MTXCAP` or `CAP.TILE` are *illegal instruction*.
* Non-canonical tile layouts for Tier-0 handoff are reserved or implementation-private and out of scope.
* Precision or memory behaviors not derived from CAP/XMEM are undefined and non-conformant.

### 1.5 Notes for Implementers (Informative)

* Use XMEM for data motion and PMASK for predication; avoid duplicating control in MTX.
* Expose capability bits (e.g., HAS_MUL/MAD/BIAS, MAX_SHAPE) via MTXCAP for software feature probing.
* Keep opcode decoding independent of precision formats to minimize ISA churn; format evolution occurs in CAP, not MTX.

---

## 2. Dependencies

This section lists architectural prerequisites and optional companions for MTX. Dependencies define the precision, memory, and control context MTX instructions consume; MTX adds no new global control planes.

- **Required:** `XPHMG_CAP` precision block (`CAP.PREC.MODE`, `CAP.PREC.ALT`, `CAP.PREC.STAT`). MTX instructions consume CAP effective precision, rounding, saturation, NaN/FTZ, quantization, and sticky/exception enables.
- **Recommended:** `XPHMG_XMEM` for LDS allocation, coherence/partitioning, streaming profiles, and descriptor-driven memory policy. Without XMEM, MTX tiles must be staged by software using standard loads/stores, and coherence/streaming controls fall back to platform defaults.
- **Optional:** `XPHMG_RT` (for hybrid RTU+NPU pipelines), `RSV` (loop control; predication via PMASK), `V` (if present). Integration with these domains relies on canonical tile layouts and shared CAP/XMEM state; no MTX-specific contracts are added.

---

## 3. Design Principles

This informative section summarizes the guiding principles behind MTX: the execution core shape, how precision is sourced, how data moves, and how MTX fits into the broader XPHMG architecture. These principles frame the normative rules in later sections.

- **Small-core, high-reuse:** a single 4x4 MAC datapath is time-multiplexed across smaller or rectangular shapes to minimize area while sustaining throughput on small tiles.  
- **Precision-neutral ISA:** opcodes do not encode format bits unless stated; all numeric behavior (PET/EW/ACCW, rounding, FTZ/SAT/NaN policy) comes from `CAP.PREC.MODE/ALT/STAT`.  
- **Descriptor-first I/O:** all data motion (cache/streaming, LDS allocation, swizzle) is handled by XMEM (`XMEM.LDG/STG/PREF`, `XLDS.*`, `XSWZ.*`) or by software; MTX opcodes compute only and rely on canonical tile layouts for interop.  
- **Low-latency per tile:** optimized for per-vertex transforms, per-pixel blocks, and small tensor tiles where control overhead and reuse dominate.  
- **Unified state consumption:** MTX reuses CAP and XMEM controls and PMASK predicates; capability bits (e.g., `MTXCAP`) describe what is implemented rather than changing semantics.

---

## 4. Architectural State & Capabilities (Normative)

This section defines the architectural CSRs, banking, and capability reporting that govern MTX. It describes how tiles are stored, selected, and advertised to software, and sets the invariants for interpreting bank contents under the CAP precision model.

### 4.1 Matrix Configuration CSRs (0x7D8-0x7DF)

This subsection lists the MTX control/status registers. It specifies their roles and the invariants for their use; register semantics are interpreted under the CAP effective precision.

| Address  | Name      | Access | Description                                                                                                           |
|----------|-----------|--------|-----------------------------------------------------------------------------------------------------------------------|
| 0x7D8    | `MTXCFG`  | RW     | Global flags: `TRANSPOSE_DEF(0)`, `BIAS_EN(1)`, `SAT_EN(2)` (hint; effective saturation is governed by `CAP.PREC.MODE`). |
| 0x7D9    | `MTXSEL`  | RW     | Active matrix bank selector (0-3).                                                                                    |
| 0x7DA    | `MTX2`    | RW     | Banked storage for 2x2 (4 elements, packed by `CAP.PREC.STAT` effective formats).                                     |
| 0x7DB    | `MTX3`    | RW     | Banked storage for 3x3 (9 elements; rectangular shapes use sub-blocks).                                               |
| 0x7DC    | `MTX4`    | RW     | Banked storage for 4x4 (16 elements).                                                                                 |
| 0x7DD    | `MTXBIAS` | RW     | Optional bias vector (2-4 elements, packed).                                                                          |
| 0x7DE    | `MTXSTAT` | RO     | Status/counters: `ops_issued`, `ops_saturated`, `ftz_events`, `last_shape`.                                           |
| 0x7DF    | `MTXCAP`  | RO     | Capabilities: `HAS_MUL`, `HAS_MAD`, `HAS_BIAS`, `MAX_SHAPE(2b)` (2x2..4x4).                                            |

> Packing and endianness: element packing and size follow the effective precision (`CAP.PREC.STAT`). Software may load via ordinary stores or use `MTX.LD`.

Invariants:

* `MTXSEL` selects the active bank for subsequent MTX operations; bank contents are interpreted per `CAP.PREC.STAT` effective state.
* `MTXCFG` provides hints only; saturation behavior is controlled by CAP effective precision.
* `MTXCAP` is read-only and advertises implemented features; software must probe `HAS_*` and `MAX_SHAPE` before issuing corresponding operations.
* `MTXSTAT` fields are observational and must not affect architectural state visible to other instructions.

### 4.2 Tile Shapes & Banking Map

This subsection defines how tile shapes are expressed and how they map to MTX banks. Shape encoding uses small immediates; bank contents are interpreted with CAP effective precision.

#### Tile Shape Immediates

MTX helper operations (transpose, epilogue, reductions) operate on a logical MxN tile expressed in terms of canonical XDL 4x4 base tiles. Shape is carried by small immediates:

- `m_blk` - number of 4-row blocks,
- `n_blk` - number of 4-column blocks.

Effective dimensions:

- `M = 4 * m_blk` rows,
- `N = 4 * n_blk` columns,

with the tile understood as a composite XDL tile (`XDL_TILE(M,N)`) built from `m_blk * n_blk` canonical 4x4 sub-tiles in row-major tile order (see XDL spec).

Notes and constraints:

* Typical encodings use 3-bit fields for `m_blk`/`n_blk` (values 1-8), allowing logical shapes up to 32x32. Exact encoding is implementation-defined if it respects:
  * `m_blk >= 1`, `n_blk >= 1`,
  * `M` and `N` are multiples of 4,
  * `(M, N)` do not exceed `CAP.TILE` maxima (`MAX_M_4X`, `MAX_N_4X`).
* If `COMPOSITE = 0` in `CAP.TILE`, only `m_blk = n_blk = 1` (4x4) is valid; other shapes are *illegal instruction*.
* If `COMPOSITE = 1`, any `(M,N)` within `CAP.TILE` maxima is valid; implementations may decompose into multiple 4x4 sub-operations internally.

#### Shapes and Banking

| Shape | Elements | Bank                |
|-------|----------|---------------------|
| 2x2   | 4        | MTX2                |
| 2x3   | 6        | MTX3 (top-left)     |
| 3x2   | 6        | MTX3 (top-left)     |
| 3x3   | 9        | MTX3                |
| 2x4   | 8        | MTX4 (first 2 rows) |
| 4x2   | 8        | MTX4 (first 2 cols) |
| 4x4   | 16       | MTX4                |

All packing (FP8/BF16/FP16/FP32, INT8/INT4) follows `CAP.PREC.STAT`.

---

## 5 Tile Model & Common Rules (Normative)

This section defines how MTX tiles are interpreted under CAP precision, how memory and residency are governed via XMEM, and the cross-domain rules MTX must follow. It sets the common semantics shared by all MTX instructions.

### 5.1 Precision Model Integration

All numeric behavior comes from **CAP.PREC**:

- Program base and alternate formats via `CAP.PREC.MODE` / `CAP.PREC.ALT`, then verify with `CAP.PREC.STAT`.  
- Fields such as `ACCW`, `SAT`, `FP_RMODE`, `DENORM_FTZ`, `Q`, `PACK` are not redefined here.  
- Implementations must execute MTX operations using the effective precision reported by `CAP.PREC.STAT`.

Invariants:

* MTX opcodes carry no independent format selection; PET/EW/ACCW and rounding/exception policy are taken from CAP effective state at the instruction boundary.
* Mixed-precision and alternate modes follow CAP MIXED/ALT rules; MTX must not introduce MTX-local numeric policy.
* Saturation/FTZ/NaN handling and sticky/exceptions are governed by CAP; MTX must not override them.

### 5.2 Data Movement & Memory (with XPHMG_XMEM)

- Use `XLDS.ALLOC/FREE/BAR/CPY` to stage tiles into LDS.  
- Use `XMEM.PREF/LDG/STG` (and `XMEM.DLOADC/DSTOREC` if enabled) to manage cache/streaming/compression.  
- For hybrid MTX-RT passes, set `XPHMG_MEMCTL.{mclass, sprof_id, hyb, dom}` in descriptors to steer partitioning/QoS (CL0..CL3).

> MTX instructions do not perform DMA.

Invariants:

* Tile residency, coherence, streaming, compression, and partitioning are controlled by XMEM descriptors and CAP.XMEM state; MTX instructions do not carry memory policy bits.
* If XMEM is absent, software must stage tiles via standard loads/stores; correctness must not depend on XMEM-specific features.

### 5.3 Cross-Domain Interoperability & Implementation Requirements (Normative)

#### 5.3.1 Precision / Numeric State (CAP -> MTX)

Every MTX instruction (`MTX.LD`, `MTX.MUL`, `MTX.MAD`, vendor variants) must snapshot the effective precision state at the instruction boundary: `CAP.PREC.MODE`, `CAP.PREC.ALT`, `CAP.PREC.STAT`, `CAP.PREC.EXC.{EN,ST}`.

From these, MTX derives: effective PET/EW and ACCW; rounding mode (`EFF_FP_RMODE`); quantization state (`EFF_Q`, ZP, SCALE_SH); NaN policy; ZMODE; SAE behavior; UNS for integer signedness.

Conformance rules:

1. MTX must not reinterpret FP or INT formats outside CAP definitions.  
2. Mixed-precision GEMMs follow CAP ALT/MIXED rules.  
3. Widening accumulation respects `EFF_ACCW`.  
4. Down-casts set `DOWNCAST_TAKEN` or `UNSUP_FMT` if outside supported domain.  

Forbidden: MTX-local rounding, NaN, quantization, or saturation rules. MTX only uses CAP effective precision semantics.  
No additional CAP-derived controls beyond those listed are consumed in v0.1; future MTX variants must enumerate any new controls explicitly.

#### 5.3.2 Predication & ZMODE Semantics (PMASK)

MTX consumes effective predication only from PMASK, per `xphmg.md`:

```
pred[i] = PMASK.bank[pbank_eff][i]
```

where `pbank_eff` is selected by an instruction `pbank` field when present or by an architectural predication prefix; if no selection is provided, `pbank = 0` (virtual ALL-ONES). MTX defines no local predicate/mask registers or implicit predicate sources.

Execution under predication is mandatory: for any element/lane/tile with `pred[i] = 0`, MTX MUST NOT execute the operation, MUST NOT read operands (source tiles, matrices, accumulators), MUST NOT raise exceptions, and MUST NOT produce side-effects (including accumulator updates, flags, or architectural state). This applies uniformly to fused-multiply-add/dot/reduction ops, accumulator updates, conversion/packing, and any operation that writes register-like destinations.

Masked writeback follows `CAP.PREC.MODE.ZMODE`:

| ZMODE | Behavior                                         |
|-------|--------------------------------------------------|
| 0     | masked elements preserve prior destination value |
| 1     | masked elements write architectural zero (per EW)|

Destination for MTX includes tile banks (`MTX2/MTX3/MTX4/MTXBIAS`), accumulator state, scalar register results, and result buffers. MTX MUST NOT define any alternate predicated writeback policy.

#### 5.3.3 FP Exceptions & Sticky Flags

MTX operations must report: `SAT_HIT`, `FTZ_HIT`, `DOWNCAST_TAKEN`, `UNSUP_FMT`, and FP stickies (`NV`, `DZ`, `OF`, `UF`, `NX`, `QNAN_SEEN`, `SNAN_SEEN`).

Trap rules: trap only if `EXC.EN[bit]=1` and `EFF_SAE=0`. If SAE override applies (CAP or RSV prefix), suppress trap but set stickies.  
First-fault: MTX does not define FOF for math pipelines; first-fault reporting uses the architecturally defined fault-reporting sink for the consuming ISA domain. For MTX memory FOF, see Section 5.3.4.

#### 5.3.4 Unified Memory Behavior (CAP.XMEM -> MTX)

All MTX memory operations (`MTX.LD`, streaming tile loads, tiled scatter/stores, LDS-resident tiles) must obey `CAP.XMEM.DOM`, `CAP.XMEM.CLSMAP`, `CAP.XMEM.LDS_BUDGET`, `CAP.XMEM.STREAM_CTL`, `CAP.XMEM.SPROF0..3`, `CAP.XMEM.COMP_CTL`, `CAP.XMEM.DESCPOL`, and all SVMEM controls.

Mandatory rules:

1. Coherence: use the domain from `CAP.XMEM.DOM` unless a descriptor overrides it.  
2. Streaming: prefetch and stride rules match RSV/GFX.  
3. Predication: for any MTX memory-visible operation, elements/lanes/tiles with `pred[i] = 0` MUST NOT issue memory accesses and MUST NOT fault; memory-side predication is governed by `XPHMG_XMEM`.  
4. SVMEM: indexed tile loads must use SVMEM, including predication, address scaling, FOF, and ZMODE for masked destinations.  
5. Compression: if enabled in `CAP.XMEM.COMP_CTL`, MTX adopts the same default.  
6. Descriptors: only explicitly set fields in `XPHMG_MEMCTL` override CAP defaults.  

FOF: If an MTX tile load uses SVMEM with FOF=1, on first fault report the index via the fault-reporting sink defined by the consuming ISA domain, abort, and allow software retry. MTX must not define a different FOF mechanism.

#### 5.3.5 Cross-Domain Dataflow Guarantees (MTX <-> RSV/RT/GFX/NPU)

* RSV: Tiles prepared by RSV (permutes/interleaves) are consumed without reinterpretation; MTX outputs are readable by RSV with identical numeric rules.  
* RT: MTX must not change NaN/FP behavior observable by ray tests; transforms used in RT preserve CAP precision.  
* GFX: MTX may serve small MLP, skinning, or 3x3/4x4 transforms; masking/ZMODE/FP behavior remain consistent.  
* NPU: MTX small tiles are numerically equivalent to NPU using the same CAP precision; software may fuse MTX/NPU GEMM paths without precision mismatch.  

#### 5.3.6 MTX Tile Layout, Addressing & LDS Rules

Tile layout rules (normative):

1. Tile element format = CAP effective PET/EW.  
2. Tile accumulator format = CAP effective ACCW.  
3. Mixed-precision tiles map to CAP.ALT rules.  
4. Tiles stored to memory must be readable by RSV/RT/GFX without reinterpretation.  

LDS allocation rules:

1. MTX tile banks allocate via `XLDS.ALLOC` using the same class budget rules as RSV/RT/GFX.  
2. Tile bank swizzle (if implemented) honors `CAP.XMEM.CAP.BIDIR_SWZ`.  
3. LDS barriers use the same fence ordering: after tile write, issue `XLDS.BAR >=CLUSTER` before visibility to other domains.  

#### 5.3.7 Descriptor Integration (`XPHMG_MEMCTL`)

If invoked through a descriptor, only explicitly set fields (`dom`, `mclass`, `sprof_id`, `hyb`, `sec`) override CAP; unspecified fields revert to CAP.XMEM defaults. Descriptors must not introduce MTX-specific semantics.

#### 5.3.8 Forbidden Divergences

MTX must not change ZMODE behavior; invent private rounding or saturation; change coherence/streaming rules; treat NaNs differently from CAP; reinterpret PET/EW/ACCW; ignore SVMEM predication or FOF; or implement alternative gather/scatter semantics. Any violation is non-conformant.

#### 5.3.9 Debug & Deterministic Replay Requirements

MTX must expose to debug: latched CAP.PREC effective state, CAP.XMEM effective state, MTXCFG/MTXSTAT, and tile bank contents if architecturally visible. Replay must see stable precision and memory state per decoded instruction; partial application of CAP or XMEM state is forbidden.

#### 5.3.10 Summary

MTX is a consumer of the unified XPHMG architectural state: precision from CAP, memory from CAP.XMEM, and mask/FOF/addressing via PMASK and XMEM SVMEM rules. MTX is not a separate subsystem.

### 5.4 Exceptions & Status

* Illegal Instruction: unsupported shape when `MTXCAP.MAX_SHAPE` is smaller; `HAS_MAD=0` with `MTX.MAD`.  
* Precision downgrade: reflected in `CAP.PREC.STAT` (`UNSUP_FMT`, `DOWNCAST_TAKEN`, `SAT_HIT`, `FTZ_HIT`).  
* `MTXSTAT` may increment `ops_saturated` and `ftz_events` for profiling.  

Notes:

* Exception/trap routing follows CAP; MTX defines no alternate routing.  
* `MTXSTAT` counters are observational only; they must not alter architected execution.  

---

## 6 Helper Operations on Tiles (Normative)

This section defines helper operations on MTX tiles (transpose, epilogue, and reductions) and the rules they must follow. All helpers inherit precision, masking, and layout semantics from Sections 4-5 and do not introduce alternate policies.

### 6.1 Tile Transpose Helper (`MTX_TRANSPOSE`)

`MTX_TRANSPOSE` transposes an MxN composite tile; both dimensions are multiples of 4 and are carried in tile-shape immediates (`m_blk`, `n_blk`).

#### 6.1.1 Scope

The instruction takes:

- A source tile `T_src` laid out in an `XDL_TILE(M,N)` layout (composed of `m_blk * n_blk` 4x4 sub-tiles in row-major tile order),
- A destination tile `T_dst` with dimensions `N x M` in the corresponding `XDL_TILE(N,M)` layout,
- Tile-shape immediates `m_blk`, `n_blk` as defined in Section 4.2.

Effective dimensions: `M = 4 * m_blk`, `N = 4 * n_blk`. The special case `m_blk = n_blk = 1` corresponds to a single 4x4 tile.

#### 6.1.2 Semantics

For all `0 <= i < M` and `0 <= j < N`:

```
T_dst[j][i] = T_src[i][j]
```

Indices `(i,j)` and `(j,i)` are in logical MxN / NxM matrix space, with `T_src` and `T_dst` stored using canonical XDL composite-tile layout. Implementations may decompose into 4x4 in-tile transposes plus sub-tile permutations if the architectural result matches the mathematical transpose.

#### 6.1.3 Valid Shapes

- If `COMPOSITE = 0` in `CAP.TILE`, only `m_blk = n_blk = 1` (4x4) is valid.
- If `COMPOSITE = 1`, any `(M,N)` allowed by `CAP.TILE` (`MAX_M_4X`, `MAX_N_4X`) is valid; larger shapes are illegal.

Tiles involved in `MTX_TRANSPOSE` must reside in XMEM regions whose layout metadata indicates an XDL tile layout; the instruction must not be applied to non-tile layouts.

### 6.2 Epilogue Stage (Bias / Clamp / Activation)

Epilogue operations act on an MxN tile, where `m_blk`, `n_blk` determine `M`, `N` (Section 4.2). The tile is stored in `XDL_TILE(M,N)` layout in XMEM. An epilogue reads `T` (the just-computed tile) and optionally a bias term and clamp thresholds, producing a new tile `T'` of the same shape and layout.

Minimal recommended semantics:

1. Bias-add: for each `i,j`, `T'[i][j] = T[i][j] + bias(i,j)` according to the selected broadcast mode (scalar / row / column / per-element).
2. Clamp: `T''[i][j] = clamp(T'[i][j], MIN, MAX)` with MIN/MAX fixed or configured via MTX configuration CSRs.

Arithmetic obeys the current CAP precision policy. Implementations may fuse the epilogue with MAC operations or execute it as a separate instruction if the visible result matches bias-add followed by clamp.

### 6.3 Tile Reductions

Reduction helpers operate on an MxN tile, dimensions defined by `m_blk`, `n_blk` (Section 4.2), stored in `XDL_TILE(M,N)` layout. The tile is logical; storage layout is canonical XDL.

Minimal recommended reductions:

1. Full-tile SUM:
```
s = sum_{i=0..M-1} sum_{j=0..N-1} T[i][j]
```
`s` is a scalar result written to a destination register or a 1x1 tile.

2. Row-wise SUM: for each row `i`,
```
r[i] = sum_{j=0..N-1} T[i][j]
```
producing an M-element vector (may be stored as 1xM or Mx1 XDL tile in XMEM).

Implementations may extend reductions to columns and other operators (MAX/MIN, etc.) optionally.

Numeric behavior follows CAP precision policy. If saturation or other exceptional events occur, CAP.PREC sticky flags must be updated accordingly. If `COMPOSITE = 0`, only 4x4 reductions are valid; otherwise, any MxN within `CAP.TILE` maxima is valid.

### 6.4 Notes for Implementers (Informative)

* Helper operations inherit all mask/ZMODE, precision, and memory-layout rules from Sections 4-5; no helper-specific overrides exist.
* For composite tiles, an implementation may decompose helpers into 4x4 primitives; architectural results must match the logical operation on the MxN tile.
* Helper opcodes use the same custom-2 opcode (`0x5B`) with dedicated `funct3` values distinct from `MTX.LD/MUL/MAD`; no additional sub-encoding space is defined in v0.1.

---

## 7 Instruction Set (Normative)

This section defines the MTX instruction forms, their operands, and required behaviors. Encodings use the custom-2 opcode space (`0x5B`). All encodings are I-type 32-bit unless noted. `SHAPE` must not exceed `MTXCAP.MAX_SHAPE`; illegal shapes trap. Precision, predication, ZMODE, and exceptions follow CAP/PMASK/XMEM rules defined in Sections 4-5. Final `funct` values remain in the vendor table.

### 7.1 `MTX.LD` - Load Matrix Bank from Memory (convenience)

This instruction loads a matrix (and optional bias) from memory into the active MTX bank. It uses I-type encoding with shape and layout hints carried in the immediate; precision comes from CAP effective state.

```
imm[11:0] | rs1 | funct3=000 | rd=x0 | opcode=01011011
```

Fields (in `imm`):
- `SHAPE[3:0]`  (2x2=0x22, 2x3=0x23, 3x2=0x32, 3x3=0x33, 2x4=0x24, 4x2=0x42, 4x4=0x44)  
- `TRANSPOSE`   (bit 4)  
- `ROWMAJOR`    (bit 5)  
- `WITH_BIAS`   (bit 6)

Semantics: load matrix elements (and optional bias) from `[rs1]` into the active bank (`MTXSEL`). Element interpretation uses the effective precision (`CAP.PREC.STAT`). Rectangular shapes map to sub-blocks of `MTX3/MTX4`. It is illegal if `SHAPE` exceeds `MTXCAP.MAX_SHAPE`.

Note: Software may also fill MTXx via ordinary `csrw` stores; `MTX.LD` is a helper.

### 7.2 `MTX.MUL` - Matrix x Vector Multiply

This instruction performs a matrix-times-vector multiply using the active bank, with optional transpose and stride hints. Shape and control bits are carried in the immediate; numeric behavior follows CAP.

```
imm[11:0] | rs1 | funct3=001 | rd | opcode=01011011
```

Fields (in `imm`):
- `SHAPE[3:0]` (same codes as above)
- `TRANSPOSE`  (bit 4) - XOR with `MTXCFG.TRANSPOSE_DEF`
- `SAT_HINT`   (bit 5) - hint; effective saturation from CAP
- `VEC_COUNT[2:0]` (bits 8:6) - number of contiguous vectors to process (1..8)
- `STRIDE_BYTES[1:0]` (bits 10:9) - 0=packed, 1=elem, 2=row, 3=impl.

Operands:
- `rs1` = base address of input vector(s)  
- `rd`  = base address for output vector(s)

Semantics: compute `u = M x v` for `VEC_COUNT` vectors, using the active bank and the effective precision. Stride hints aid prefetch; correctness must not depend on them. It is illegal if `SHAPE` exceeds `MTXCAP.MAX_SHAPE` or `HAS_MUL=0`.

### 7.3 `MTX.MAD` - Matrix x Vector + Bias (optional)

This instruction performs matrix-times-vector with bias add, sourcing bias from `MTXBIAS` when supported by `MTXCAP.HAS_MAD`. Shape and control bits mirror `MTX.MUL`; numeric behavior follows CAP.

```
imm[11:0] | rs1 | funct3=100 | rd | opcode=01011011
```

Same as `MTX.MUL`, with bias from `MTXBIAS`: `u = (M x v) + bias`. Presence is indicated by `MTXCAP.HAS_MAD`. It is illegal if `HAS_MAD=0` or `SHAPE` is unsupported.

### 7.4 Canonical Tile Output Layout

All MTX instructions that write tile results into XMEM must produce data in canonical XDL layout:

- 4x4 row-major tiles (`XDL_TILE4x4_ROWMAJOR`), or  
- Valid composite-tile arrangements (`XDL_TILE(M,N)` composed of multiple 4x4 tiles).

This layout is required for Tier-0 cross-engine handoff. MTX must not use implementation-private tile encodings when writing into Tier-0.

Linking behavior: when targeting a composite tile, implementations may compute only required 4x4 tiles or fuse multiple tiles; architectural results must match the composite-tile definition.

### 7.5 Encodings (summary)

| Instruction | I-Type (32b) | Description |            |       |       |                        |
| ----------- | ------------ | ----------- | ---------- | ----- | ----- | ---------------------- |
| `MTX.LD`    | `imm         | rs1         | funct3=000 | rd=x0 | 0x5B` | Load matrix bank       |
| `MTX.MUL`   | `imm         | rs1         | funct3=001 | rd    | 0x5B` | Matrix x vector        |
| `MTX.MAD`   | `imm         | rs1         | funct3=100 | rd    | 0x5B` | Matrix x vector + bias |

> Exact `imm` bit allocation is defined in Sections 7.1/7.2.

---

## 8 Conformance & Compatibility (Normative)

This section defines how MTX reports its capabilities and how it aligns with the broader XPHMG environment. It constrains feature discovery and clarifies what is in scope for compatibility.

### 8.1 Conformance & Capabilities

This subsection defines how implementations report MTX capabilities and how software interprets `HAS_*` and `MAX_SHAPE` for feature gating.

Conformance and reporting rules:

* `MTXCAP.HAS_MUL` and `MTXCAP.HAS_MAD` advertise presence of `MTX.MUL` and `MTX.MAD`. If a bit is 0, the corresponding instruction is *illegal instruction*.
* `MTXCAP.HAS_BIAS` advertises whether bias storage (`MTXBIAS`) is implemented; if 0, bias-dependent forms are *illegal instruction*.
* `MTXCAP.MAX_SHAPE` encodes the largest supported tile (2x2..4x4). Any `SHAPE` immediate exceeding this is *illegal instruction*.
* Capability bits are read-only and must reflect static implementation features; they are not affected by CAP/XMEM state.
* Software must probe `MTXCAP` before issuing profile-dependent code paths; fallback or feature gating is required if a capability is absent.

### 8.2 Compatibility

Compatibility describes how MTX coexists with other XPHMG components and legacy paths. It does not add MTX-specific modes beyond CAP/XMEM.

- Precision control for MTX is provided through `CAP.PREC.*`.
- MTX data loads/stores are defined through standard memory operations and `XPHMG_XMEM` + LDS; bespoke `XREG` paths and legacy `XMEM.DMA`/`XMEM.LDGPR/STGPR` opcodes are outside this specification.

---

## 9 Scheduler & Hybrid Interop (informative)

This informative section outlines scheduling patterns and descriptor choices when MTX is combined with RT, GFX, and RSV-controlled loops. It provides guidance on QoS classes, barriers, and descriptor fields that preserve correctness under the CAP/XMEM rules defined earlier. No new architectural controls are introduced; implementations remain free to reorder internally so long as architectural visibility and CAP/XMEM contracts are respected.

Typical hybrid pipeline:

1) RTU builds G-Buffer (class CL0),  
2) MTX denoiser or MLP pass (class CL1),  
3) RTU composite (CL0 or HYB CL2).

Drivers should set `memctl.mclass={0,1,2}`, `memctl.sprof_id` (BVH vs weights profile), and `memctl.hyb` for shared phases. Barriers: `XFENCE.SIGNAL/WAIT` + `XLDS.BAR >= CLUSTER` for LDS/L1/L2 visibility.

Scheduler guidance (informative):

- **QoS and classes:** Use `memctl.mclass` to partition bandwidth between RT (CL0/CL2) and MTX (CL1). MTX bursts should honor the XMEM streaming profile indicated by `memctl.sprof_id` to coexist with RTu traffic.  
- **Descriptor staging:** For shared LDS tiles between MTX and RT/GFX, emit `XLDS.BAR` at least at CLUSTER scope after MTX writes before RT/GFX reads; follow with `XFENCE.SIGNAL/WAIT` if cross-engine visibility is required.  
- **Hybrid phases:** When `memctl.hyb=1`, MTX and RTu may interleave on the same tile buffers; software should sequence producer/consumer passes with fences rather than relying on implicit ordering.  
- **Latency hiding:** MTX tile loads may be prefetched via `XMEM.PREF` or `XLDS.CPY` while RTu is active, provided class/QoS separation is maintained; correctness must not depend on speculative overlap.  
- **Debug/replay:** Deterministic replay requires the same CAP/XMEM state and descriptor fields observed in Section 5.3; schedulers should avoid time-varying descriptor patches mid-frame unless paired with explicit fences.

---

## 10 Programming Examples

This informative section provides worked sequences that demonstrate how MTX instructions interact with CAP precision state and XMEM/LDS staging. Each example shows the minimum setup for precision, bank selection, tile load, and compute. The sequences are non-normative and do not alter architectural semantics; they illustrate expected ordering and control usage.

### 10.1 2x3 affine (FP16 base, ACCW=FP32)

This example shows precision setup, bank load, and a simple matrix-vector multiply on a 2x3 shape.

```asm
# Precision (CAP.PREC)
li  t0, ( (0b10<<22) /*ACCW=FP32*/ | (0b011<<19) /*PET=FP16*/ | (0b01<<17) /*EW=16*/ | (1<<31) )
csrw 0x7D0, t0   # CAP.PREC.MODE (APPLY0=1)

# Load matrix bank 0 from memory (row-major 2x3)
csrw 0x7D9, x0   # MTXSEL=0
li   a0, mtx23_ptr
MTX.LD  imm=(0x23 | (1<<5)), rs1=a0

# Multiply 64 vectors, packed stride
li   a1, vin_ptr
li   a2, vout_ptr
MTX.MUL imm=(0x23 | (0<<4) | (0<<5) | (0b111<<6) | (0<<9)), rs1=a1, rd=a2
```

### 10.2 FP8 E4M3 mixed (mul FP8, acc FP16), 4x4

This example uses an alternate precision mode to perform a 4x4 transform with FP8 inputs and FP16 accumulation.

```asm
# Alternate precision
li  t1, (1<<30) /*ALT_EN*/ | (0b0010<<26) /*FP8_E4M3*/ | (0b01<<24) /*ALT_ACCW=FP16*/ | (1<<22) /*MIXED*/ | (1<<31)
csrw 0x7D1, t1   # CAP.PREC.ALT (APPLY1=1)

# 4x4 transform of 8 vectors (contiguous)
MTX.MUL imm=(0x44 | (0<<4) | (0<<5) | (0b111<<6) | (0<<9)), rs1=a1, rd=a2
```

### 10.3 INT8xINT8 with zero-point (Q path enabled)

This example enables quantization in base mode and performs an INT8 matrix-vector multiply.

```asm
# Base mode: quant enabled
li  t0, (1<<25) | (1<<31)    # Q=1, APPLY0
csrw 0x7E0, t0
# Alt: INT4 or leave INT8 as base (example leaves INT8)
MTX.MUL imm=(0x33), rs1=a1, rd=a2
```

---

## 11 Hardware Notes (Informative)

This section provides non-normative guidance on microarchitectural choices and pipeline behavior. It does not add architectural requirements; any implementation freedom exercised here must preserve the architectural semantics defined in preceding sections, including CAP precision rules, XMEM residency, and canonical tile layout.

* Datapath: a single 4x4 core may implement FMA for FP modes and MAC for INT modes, time-multiplexed across composite tiles; internal decomposition must not alter architectural results.
* Accumulators: ACCW widens the accumulator pipe (e.g., FP32 or INT32) independent of input EW; rounding/FTZ/SAT detection must still follow CAP timing at the instruction boundary even if the pipe is deeper.
* Shapes: rectangular shapes may gate unused lanes; bias add (if implemented) occurs in the accumulator stage after MAC/FMA and before any down-cast.
* Issue/latency: back-to-back issues are permissible when operands are resident in LDS/cache; stall insertion and prefetch are handled by XMEM. Implementations may coalesce tile loads or bypass banks internally if the architectural register/memory view is preserved.
* Banking/conflicts: MTXSEL selects the visible bank; internal banking or double-buffering is implementation-defined but must honor MTXCFG/MTXSTAT visibility and the active bank observable by software.
* Exceptions/stickies: FP and saturation flags are latched from CAP at retire; pipeline optimizations must not drop stickies or change which operations trigger `SAT_HIT` or `FTZ_HIT`.
* Determinism/replay: debug or replay features should capture the effective CAP and XMEM state at decode/issue so that re-execution under the same state produces identical architectural results.

---

*CC-BY-SA 4.0 -- Open Specification maintained by Pedro H. M. Garcia.*  
Designed to integrate with `XPHMG_CAP`, `XPHMG_XMEM`, `XPHMG_RT`, and optional `RSV` or `V`.
