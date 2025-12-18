# RISC-V Vendor Extension **XPHMG_XMEM**
**Unified LDS / Cache / Streaming Memory Control**

**Category:** Vendor Extension (part of `XPHMG_*` family)  
**Version:** 0.1.0
**Author:** Pedro H. M. Garcia  
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview (Informative)

`XPHMG_XMEM` defines the explicit memory control model shared by all XPHMG execution domains. It provides architectural control for LDS allocation, cache/streaming behavior, memory classes, coherence domains, indexed gather/scatter (SVMEM), and descriptor-driven overrides. Configuration is sourced from CAP-owned CSRs; mirrors are optional.

### 1.1 Conceptual Model

- XMEM supplies configuration for instructions defined elsewhere; it does not alter their semantics. Fields include classes, domains, streaming profiles, compression defaults, and gather/scatter parameters.
- CAP (`CAP.XMEM.*`) is the sole architectural source of truth. Mirrors (XMIR.*) may exist for performance but must match CAP at instruction boundaries.
- Instructions and descriptors consume CAP defaults unless a non-zero, supported override is provided.
- RSV, MTX, RT, GFX, ACCEL, and CPU observe the same classes, domains, tiers, and SVMEM rules.

### 1.2 Architectural Guarantees and Invariants

- State is latched at instruction boundaries; mid-instruction CSR writes have no effect on the in-flight operation.
- Class and domain identifiers are uniform across engines; engines must not reinterpret them.
- Capability gating: if a bit in `CAP.XMEM.CAP` is clear, dependent controls are RAZ/WI and behavior is disabled.
- Descriptor fields override CAP defaults only when non-zero and supported; otherwise CAP defaults apply.
- Visibility follows CAP-selected domains and scopes; fences order with respect to those domains.

### 1.3 Interaction with Other XPHMG Extensions

- CAP defines the authoritative CSRs, feature bits, defaults, and visibility scopes consumed by XMEM instructions and descriptors.
- RSV/MTX/RT/GFX/ACCEL reuse XMEM for LDS allocation, cache/streaming hints, SVMEM gather/scatter, and class/domain selection; no extension redefines XMEM fields.
- PREC supplies element width and predication context for SVMEM (e.g., `EFF_EW`, `ZMODE`); XMEM gather/scatter honors these controls.
- XDL provides layout metadata used by multiple engines; tiers describe placement/latency, not layout.

### 1.4 Undefined or Reserved Behavior

- If an implementation does not support a documented feature bit in `CAP.XMEM.CAP`, related fields are RAZ/WI and have no architectural effect.
- Cross-engine register forwarding is not defined; cross-engine dataflow must use XMEM-managed memory.
- NOTE: Additional mirror behaviors beyond aliased or read-only are unspecified in v0.1 and may be clarified in a future revision.

### 1.5 Notes for Implementers (Informative)

- Program CAP; treat mirrors as optional accelerators that must be coherent at instruction boundaries.
- Document implemented `CAP.XMEM.CAP` bits; leave unimplemented fields RAZ/WI.
- Surface CAP defaults and descriptor precedence explicitly in firmware and toolchains.

---

## 2. Conventions & Terminology (Informative)

This section defines shared terms aligned with RISC-V specification style.

### 2.1 Domains and Scopes

- **Domain (`dom`)**: coherence/visibility grouping selected from `CAP.XMEM.DOM` or descriptor override (e.g., `DOM_CPU`, `DOM_RT`, `DOM_MTX`, `DOM_ALL`).
- **Scopes**: ordering hierarchy `THREAD < WARP < WG < CLUSTER < SYSTEM`; fence scopes refer to this hierarchy.

### 2.2 Classes and Partitioning

- **Memory Class (`mclass`)**: class index CL0..CL3 mapping to ways/QoS via `CAP.XMEM.CLSMAP`; classes are global and not engine-specific.
- **Active Class**: effective class when `CLASS_PART=1`; descriptor `mclass` if non-zero else `CAP.XMEM.L1CFG.ACTIVE_CLASS`.
- **Partitioning**: `XMEM.CCTL.PARTITION` or class mapping defines class-to-way/region mapping; defaults reside in CAP.XMEM.

### 2.3 Tiers

- **Tier (`tier`)**: placement/latency hint in `XPHMG_MEMCTL`:
  - `T0`: Tier-0 / shared hot data
  - `T1`: LDS/XMEM working sets
  - `T2`: coherent cache hierarchy residency
  - `T3`: DRAM / high-latency memory
- Tier semantics apply uniformly; only `tier` and `mclass` drive placement hints.

### 2.4 SVMEM Terminology

- **SVMEM**: indexed gather/scatter configured by `CAP.XMEM.SVMBASE/SVMIDX/SVMSCL/SVMLEN/SVMMODE`.
- **FOF**: fault-only-first via `SVMSCL.FOF`; first faulting element index written to `RSV.SVFAULTI`, operation aborts.
- **ZMODE**: masked-lane policy from `CAP.PREC.MODE.ZMODE`, applied to gathers/scatters.

### 2.5 Configuration Sources

- **CAP.XMEM**: source of memory classes, domains, streaming profiles, compression defaults, descriptor policy, and SVMEM config.
- **XMIR.* (mirrors)**: optional; must match CAP at instruction boundaries.
- **Descriptor (`XPHMG_MEMCTL`)**: per-job overrides for class, domain, streaming profile, security tag, LDS hints; non-zero fields override CAP defaults.

### 2.6 Notation

- CSR names follow `CAP.XMEM.*` and `XMIR.*`.
- Bit ranges use RISC-V notation (e.g., `bits[7:0]`).
- RAZ/WI indicates read-as-zero / write-ignored when a feature is disabled or unimplemented.

### 2.7 Notes for Implementers (Informative)

- Keep definitions of `dom`, `mclass`, scopes, and tiers consistent across engines.
- Document implemented `CAP.XMEM.CAP` bits; treat unimplemented as RAZ/WI.
- TODO: Clarify interaction if future revisions add domains or class ranges.

---
## 3. Architectural Memory Model

This section describes the memory hierarchy, tiers, and visibility rules. Presence of levels/features is advertised via `CAP.XMEM.CAP`; absent features are RAZ/WI.

### 3.1 Memory Hierarchy

Implementations may expose any subset of:

| Level            | Description                                                                                                    |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| **L0-LDS**       | Local Data Store (SLM). Banked SRAM per cluster, dynamically allocated via `XLDS.ALLOC`.                       |
| **L1$-Cluster**  | Unified set-associative cache with optional way-partitioning and line locking.                                 |
| **L2$-Fabric**   | Shared cache across clusters. Optionally coherent with CPU.                                                    |
| **XDMA Engines** | Streaming, non-temporal DMA with optional inline decompression/packing.                                        |
| **UMF Fabric**   | Interconnect providing domain tags (`CPU`, `RT`, `MTX`, `ALL`).                                                |

**Coherence scopes:** `THREAD < WARP < WG < CLUSTER < SYSTEM`

All fences and LDS barriers honor these scopes.

Notes (informative):
- The scope hierarchy is the only architecturally observed ordering; finer internal levels are allowed but must respect these scopes.
- If a level is absent per `CAP.XMEM.CAP`, related controls are RAZ/WI; software must not assume presence.

### 3.2 Tier Model

Tiers are placement/latency hints in `XPHMG_MEMCTL.tier`. Tiers do not alter layout; they describe expected residency and latency.

#### 3.2.1 Unified XMEM View

All engines view XMEM through `XPHMG_MEMCTL`. Direct engine-to-engine register forwarding is not defined; cross-engine dataflow must use XMEM regions.

#### 3.2.2 Memory Class for Hot Shared Data (`CLS_SHARED`)

Implementations must reserve at least one class ID for shared hot data:
- **`CLS_SHARED`** identifies regions intended for cross-engine Tier-0 communication.
Other classes remain implementation-defined (e.g., `CLS_GBUF`, `CLS_MTX`, `CLS_RT`).

#### 3.2.3 Tier Field (`tier`) in `XPHMG_MEMCTL`

`tier` (min. 2 bits) indicates intended residency:

| tier | Meaning                                  |
| ---- | ---------------------------------------- |
| `T0` | Tier-0 / shared hot data                 |
| `T1` | Regular LDS/XMEM                         |
| `T2` | Coherent cache hierarchy                 |
| `T3` | DRAM / external memory                   |

A region marked `CLS_SHARED / T0` is eligible for cross-engine handoff.

#### 3.2.4 Logical Ownership (`dom`)

`dom` indicates current owner or priority:
- `DOM_MTX`, `DOM_RT`, `DOM_GFX`, `DOM_RSV`, `DOM_ACCEL`, `DOM_CPU`, or `DOM_ALL`.
Changing `dom` on `CLS_SHARED / T0` does not imply physical movement; it is a logical transition. Default `dom` is `CAP.XMEM.DOM` unless overridden; fence scopes in Section 7 apply.

#### 3.2.5 Partitioning and Tier-0 Mapping (`XMEM.CCTL.PARTITION`)

`XMEM.CCTL.PARTITION` describes physical subranges reserved for Tier-0. Only `tier = T0` regions may map into Tier-0. Tier-0 regions must provide lower latency than Tier-1 and may impose size/aliasing/hazard constraints.

#### 3.2.6 Handoff Semantics

Handoff between engines (e.g., MTX->RT):
```
1. Producer writes region R: CLS_SHARED / T0 / DOM_PRODUCER
2. Producer or scheduler sets R.dom = DOM_CONSUMER
3. Consumer reads R; no data movement occurs
```
No additional barriers are implied beyond normal XMEM ordering unless specified by the consuming ISA extension.

#### 3.2.7 Engine Behavior

- Engines must accept `CLS_SHARED / T0` regions with matching `dom` as valid sources/sinks.
- Engines must not access Tier-0 regions without `dom` authorization.
- Engines may optimize Tier-0 accesses; architectural results must be unchanged.

#### 3.2.8 Fallback to Tier-1

If Tier-0 capacity is insufficient, engines may allocate `CLS_SHARED / T1 / DOM_*` regions. These participate in the unified dataflow model but may require explicit synchronization and have higher latency.

#### 3.2.9 Tier-1 Regions and Working Sets

Tier-1 regions hold working sets larger than Tier-0 but latency-sensitive.
- Follow standard XMEM ordering rules; subject to `dom` permissions.
- Changing `dom` on Tier-1 does not imply hot-handoff semantics; it is a policy hint.

##### 3.2.9.1 Relationship to Tier-0 and Tier-2+

- Tier-0: shared hot data for cross-engine handoff.
- Tier-1: larger working sets with LDS/XMEM latency.
- Tier-2+: coherent caches/DRAM not directly controlled by XPHMG.
Typical pattern: construct/transform in Tier-1 -> project subset to Tier-0 -> write back to Tier-1 or higher tiers.

##### 3.2.9.2 Partitioning and QoS

`XMEM.CCTL.PARTITION` may describe Tier-1 partitioning across classes/engines and QoS hints. These are implementation-defined and must not change architectural semantics.

#### 3.2.10 XDL Tiles in Tier-0 and Tier-1

If a region is `CLS_SHARED / T0` and declares `XDL_TILE4x4_ROWMAJOR` or composite layout, engines must interpret it accordingly. Tier-0 is for hot tiles; Tier-1 is preferred for constructing/linking composite tiles. Engines must not reinterpret declared layouts.

#### 3.2.11 Tier-2 and Tier-3 Regions

Tier-2/3 describe servicing by higher memory system:
- **T2**: expected residency in coherent cache hierarchy.
- **T3**: expected residency in external DRAM/high latency.

T2 regions use normal cacheable XMEM operations and benefit from coherence/locality. T3 regions are typically accessed via streaming/non-temporal paths and may bypass caches. Both remain in the coherent address space unless implementation defines otherwise. T2 vs T3 are performance hints only; loads/stores must remain globally visible and coherent per CPU ordering rules. Mapping to levels in Section 3.1 is implementation-defined. NOTE: If a coherence fabric is absent, ordering scopes still apply; performance is implementation-defined.

### 3.3 Addressing & Visibility Rules

- Default `dom` and `mclass` come from `CAP.XMEM.DOM` and `CAP.XMEM.CLSMAP`; non-zero descriptor fields override.
- Fences/barriers order effects per Section 3.1 scopes and current `dom`. Mid-instruction CSR writes do not affect in-flight ops.
- Gather/scatter uses `CAP.PREC.STAT.EFF_EW` when `SVMLEN=0`; masked lanes follow `CAP.PREC.MODE.ZMODE`. Bounds checking via `SVMMODE` is per-element.
- Streaming/compression hints come from `CAP.XMEM.STREAM_CTL/SPROF*` and `CAP.XMEM.COMP_CTL` unless a non-zero descriptor override is present; hints must not alter correctness.
- Addressing follows the host ISA; domains/classes affect visibility/placement, not address spaces.

### 3.4 Notes for Implementers (Informative)

- Treat tiers as performance guidance; correctness must not depend on mapping tiers to physical levels.
- Keep `dom`/`mclass` meaning uniform; avoid engine-local reinterpretation.
- When a capability bit is clear, expose related CSRs as RAZ/WI and keep mirrors coherent.

---

## 4. Architectural State

CAP-owned CSRs are authoritative; mirrors are optional cached views that must match CAP at instruction boundaries.

### 4.1 CAP.XMEM CSRs

#### 4.1.1 Architectural Defaults and Effective State

Unless overridden by a descriptor or instruction immediate, engines use:
- `CAP.XMEM.L1CFG`, `CAP.XMEM.L2CFG` - cache line size, replacement, locking, active class
- `CAP.XMEM.CLSMAP`, `CAP.XMEM.LDS_BUDGET` - class/way mapping and LDS soft budgets
- `CAP.XMEM.DOM` - default coherence domain
- `CAP.XMEM.STREAM_CTL`, `CAP.XMEM.SPROF[0..3]` - streaming detector/profiles
- `CAP.XMEM.COMP_CTL` - inline-compression defaults
- `CAP.XMEM.DESCPOL` - descriptor fallback policies
- `CAP.XMEM.SVMBASE/SVMIDX/SVMSCL/SVMLEN/SVMMODE` - gather/scatter control
- `CAP.XMEM.EVENTS` - sticky status (W1C)

Notes:
- If a `CAP.XMEM.CAP` bit is 0, the corresponding behavior is disabled; the CSR reads as zero and ignores writes (RAZ/WI).
- Descriptor fields (`mclass`, `sprof_id`, `dom`, `sec`) override CAP defaults only when non-zero and supported.
- Effective state is latched at instruction boundaries; mid-instruction writes have no effect on in-flight operations.

### 4.2 Optional Engine-Local Mirrors (XMIR.*) and Synchronization

Mirrors are optional CSRs for fast reprogramming or debug. They must reflect `CAP.XMEM.*` at every instruction boundary. `CAP.XMEM.*` is the single source of truth.

**N-0 (Mirror Coherence).** Immediately before any memory instruction issues, the mirror must equal the corresponding `CAP.XMEM.*` value. With deferred apply, mirrors must synchronize before the next instruction with side effects.

Implementations may choose:
- **Aliased Mirror Mode:** XMIR writes propagate to CAP (architecturally equivalent view).
- **Read-Only Mirror Mode:** XMIR writes are RAZ/WI; CAP remains authoritative.
NOTE: Other mirror behaviors are unspecified in v0.1 and may be clarified later.

#### 4.2.1  XMIR.LDSZ (RO)
Mirror of `CAP.XMEM.PARAMS.LDS_KiB`; reports LDS bytes per cluster.

#### 4.2.2  XMIR.LDSPART (RW)
Snapshot of active LDS allocations; read-only if hardware auto-partitions.

#### 4.2.3  XMIR.L1CFG (RW)
| Bits | Field        | Description                                       |
| :--: | :----------- | :------------------------------------------------ |
| 7:0  | `WAYS`       | Associativity (hint if fixed)                     |
| 15:8 | `LINE_SZ`    | Sector size (0=32 B, 1=64 B, 2=128 B, 3=impl.)    |
|19:16 | `REPL`       | Replacement (0 LRU - 1 PLRU - 2 LFU - 3 RR/impl.) |
|  20  | `LOCK_EN`    | Enable line/way lock ops                          |
|31:24 | `ACTIVE_CLS` | Active class index (0-3)                          |

#### 4.2.4  XMIR.CLSMAP (RW)
| Bits  | Field         | Description      |
| ----- | ------------- | ---------------- |
| 7:0   | `CL0_WAYMASK` | L1$ ways for CL0 |
| 15:8  | `CL1_WAYMASK` | L1$ ways for CL1 |
| 23:16 | `CL2_WAYMASK` | L1$ ways for CL2 |
| 31:24 | `CL3_WAYMASK` | L1$ ways for CL3 |
| 35:32 | `CL0_QOS`     | QoS weight       |
| 39:36 | `CL1_QOS`     | QoS weight       |
| 43:40 | `CL2_QOS`     | QoS weight       |
| 47:44 | `CL3_QOS`     | QoS weight       |

Default mapping recommendation: CL0 = RT, CL1 = MTX, CL2 = HYB, CL3 = OTHER.

#### 4.2.5  XMIR.LDS_BUDGET (RW)
| Bits  | Field     | Description       |
| ----- | ----------| ----------------- |
| 15:0  | `CL0_KiB` | Budget for CL0    |
| 31:16 | `CL1_KiB` | Budget for CL1    |
| 47:32 | `CL2_KiB` | Budget for CL2    |
| 63:48 | `CL3_KiB` | Budget for CL3    |

Overflow sets `CAP.XMEM.EVENTS.BUDGET_OVF`.

#### 4.2.6  XMIR.L2CFG (RW)
Victim policy, QoS, snoop coherence, compression threshold.

#### 4.2.7  XMIR.DOM (RW)
Default domain: `1=CPU  2=RT  3=MTX  15=ALL`.

#### 4.2.8  XMIR.STREAM_CTL (RW)
Fields: `stride`, `pf_depth`, `nontemp_en`.

#### 4.2.9  XMIR.SPROF[0-3] (RW)
Each profile: `stride[15:0]`, `pf_depth[23:16]`, `nt_policy[27:24]`.

#### 4.2.10  XMIR.COMP_CTL (RW)
Compression mode/level/dictionary for inline compression.

#### 4.2.11  XMIR.DESCPOL (RW)
| Bits | Field            | Description                         |
| ---- | ---------------- | ----------------------------------- |
| 1:0  | `DEF_CLASS`      | Default class if `mclass`=0         |
| 2    | `HYB_ENABLE`     | Allow HYB (CL2) borrowing           |
| 5:3  | `HYB_BORROW_QOS` | Borrow priority                     |
| 6    | `STRICT_DOM`     | Reject domain mismatch              |
| 7    | `RSV`            | Reserved                            |

#### 4.2.12  XMIR.EVENTS (W1C)
Sticky flags: `INV_MISS`, `LOCK_OVF`, `LDS_OOM`, `DMA_ERR`, `COMP_ERR`, `BUDGET_OVF`, `SEC_VIOL`, `HYB_FALLBK`.

### 4.3 Descriptor `XPHMG_MEMCTL`

#### C structure
```c
struct XPHMG_MEMCTL {
  uint8_t  dom;         // DOM_CPU=1, DOM_RT=2, DOM_MTX=3, DOM_ALL=15
  uint8_t  l1_hint;     // {LOCK,STREAM,NONTEMP} | waymask bits
  uint8_t  l2_hint;     // {PREFONLY,BYPASS,COMP_EN} | QoS bits
  uint8_t  lds_hint;    // {ALLOC_LOG2,BANK_SWZ,MCAST}
  uint16_t line_sz;     // 32/64/128
  uint16_t qos;         // Priority or stall budget
  uint64_t lds_softpin; // Best-effort LDS bytes to pin

  // Additional behavior
  uint8_t  mclass;      // Memory class index 0-3 (maps to xmem_clsmap)
  uint8_t  sprof_id;    // Streaming profile id 0-3
  uint8_t  hyb;         // Hybrid/shared job (1=HYB)
  uint8_t  sec;         // Security tag if SEC_DOM=1
  uint32_t rsv0;        // Reserved
};
```

#### Semantics

- `mclass` maps to CL0-CL3.
- `sprof_id` selects streaming profile 0-3.
- `hyb=1` hints hybrid job (prefer CL2).
- `sec` ignored unless `SEC_DOM=1`.
- Unsupported fields are ignored safely.

---
## 5. Execution Model

This section describes how XMEM-controlled behavior is applied during execution. It relies on CAP state and the scope/dom/class model in Sections 2-4.

### 5.1 SVMEM Gather/Scatter Semantics

SVMEM defines indexed loads/stores governed by CAP and optional descriptor overrides, shared across all engines issuing indexed operations.

#### 5.1.1 Index Addressing Model

- Control sources: `CAP.XMEM.SVMBASE`, `...SVMIDX`, `...SVMSCL`, `...SVMLEN`, `...SVMMODE`.
- Element size: if `SVMLEN=0`, use `CAP.PREC.STAT.EFF_EW`; else `SVMLEN`.
- Scale & sign: `SVMSCL.SCALE` in {1,2,4,8,16}; `SVMSCL.SIGNED` selects sign-extend.
- Gather vs. scatter: `SVMSCL.GS` selects load or store.
- FOF: if `SVMSCL.FOF=1`, on first faulting element i, write i to `RSV.SVFAULTI` and abort.
- Bounds: if `SVMMODE.BOUND_CHECK=1`, apply per element; if `ALLOW_OOB_ZERO=1`, out-of-range elements are zeroed instead of faulting.

#### 5.1.2 Predication and Masked Lanes

- Per-lane requests are enabled by the current RSV mask.
- Masked lanes follow `CAP.PREC.MODE.ZMODE`: zeroing for gathers; scatters do not store for masked lanes.

#### 5.1.3 Address Computation
```
addr[i] = SVMBASE + sext_or_zext(IDX[i]) * SCALE
```
Ordering: each element follows current domain/scope; mid-instruction CSR changes do not affect in-flight elements.

#### 5.1.4 Memory Ordering and Domains

- Default domain is `CAP.XMEM.DOM` unless overridden by descriptor or instruction hint.
- Visibility is governed by fences (Section 7); engines must honor domain-scope fences.

Notes:
- SVMEM semantics are uniform across engines; no engine-specific reinterpretations of `SVMSCL`, `SVMMODE`, or FOF.
- Fault-only-first writes the first faulting element index and terminates the instruction; later elements do not issue.

### 5.2 LDS Model

- Allocation/Free: `XLDS.ALLOC`/`XLDS.FREE` unchanged; when `CLASS_PART=1`, allocation uses the effective class (`mclass` override else `ACTIVE_CLASS`). Failure returns 0 without state change.
- Budgets: if `CAP.XMEM.LDS_BUDGET` is non-zero, apply best-effort enforcement; overflow sets `CAP.XMEM.EVENTS.BUDGET_OVF`.
- Barriers: `XLDS.BAR` orders LDS ops up to the requested scope (Section 3.1); all engines must honor LDS barriers.
- Atomics: `XLDS.ATOMS` follow standard atomic semantics within LDS; ordering per requested scope.
- Swizzle/Copy: `XLDS.CPY` may swizzle bidirectionally if `CAP.XMEM.CAP.BIDIR_SWZ=1`; correctness must not depend on swizzle.

Notes:
- LDS is not implicitly coherent with caches; cross-domain visibility requires fences/barriers.
- Class partitioning affects allocation source but not correctness; failures are reported via EVENTS or zero return.

### 5.3 Cache/Streaming Behavior

Configured via CAP with optional descriptor overrides; affects placement/performance, not correctness.

- Class mapping: `CAP.XMEM.CLSMAP` or `XMEM.CCTL PARTITION` (when `CLASS_PART=1`) defines way/QoS mapping; class IDs are uniform across engines.
- Streaming profiles: defaults from `CAP.XMEM.STREAM_CTL` and `CAP.XMEM.SPROF[0..3]`; descriptor `sprof_id` overrides when non-zero.
- Compression defaults: from `CAP.XMEM.COMP_CTL`; descriptor or opcodes may override when supported.
- Hints: `{LOCK, STREAM, NONTEMP, PREFONLY, BYPASS}` influence placement/prefetch only; they must not change architectural data or ordering.

Notes:
- If a feature is disabled in `CAP.XMEM.CAP`, related controls are RAZ/WI; instructions must still execute correctly.
- Implementations may realize hints differently; architectural results must match.

---

## 6. Instruction Set Overview

This section summarizes instruction families that consume XMEM configuration. Semantics and correctness are governed by CAP state and descriptor overrides; hints affect placement/performance only.

- Allocation/Free: `XLDS.ALLOC`/`XLDS.FREE` unchanged; when `CLASS_PART=1`, allocation uses the caller's active class (descriptor `mclass` if non-zero, else `CAP.XMEM.L1CFG.ACTIVE_CLASS`).
- Budgets: if `CAP.XMEM.LDS_BUDGET` is non-zero, apply best-effort enforcement; overflow sets `CAP.XMEM.EVENTS.BUDGET_OVF`.
- Bank swizzle: if `CAP.XMEM.CAP.BIDIR_SWZ=1`, `XLDS.CPY` may swizzle both directions as a hint; correctness must not rely on swizzle.

### 6.1 LDS instructions (`XLDS.*`)

| Mnemonic                          | Summary                                                                         |
|-----------------------------------|---------------------------------------------------------------------------------|
| **XLDS.ALLOC rd, rs_size, flags** | Allocate LDS space; return base or 0 on failure.                                |
| **XLDS.FREE rs_base**             | Release allocated LDS region.                                                   |
| **XLDS.BAR imm_scope**            | Barrier ordering LDS ops up to given scope.                                     |
| **XLDS.ATOMS op, [addr], val**    | Atomic ops on LDS.                                                              |
| **XLDS.CPY rd, rs, len, flags**   | Copy between LDS and regs/global. If `BIDIR_SWZ=1`, swizzle in both directions. |

Notes:
- `XLDS.BAR` scopes map to Section 3.1; order relative to the specified scope and current domain.
- `XLDS.ALLOC` failure returns 0 without modifying state.
- `XLDS.ATOMS` follow standard atomicity; ordering per requested scope.

### 6.2 Cache & streaming (`XMEM.*`)

| Mnemonic                        | Summary                                                             |
|---------------------------------|---------------------------------------------------------------------|
| **XMEM.LDG rd, [addr], hint**   | Load with hint (L1_ONLY, L2_ONLY, LDS_PREF, NONTEMP, LOCK, STREAM). |
| **XMEM.STG [addr], rs, hint**   | Store with hint.                                                    |
| **XMEM.PREF [addr], len, hint** | Prefetch (L1/L2/STREAM). Non-faulting.                              |
| **XMEM.CCTL op, arg0, arg1**    | Cache control ops (INV, FLUSH, LOCK, UNLOCK, PARTITION).            |
| **XMEM.STREAM cfg0, cfg1**      | Program streaming detector or profile.                              |

**CCTL PARTITION behavior:**
- When `CLASS_PART=1`, accepts `class_id, waymask` to reprogram a class mapping (`xmem_clsmap`).
- Otherwise behaves as legacy range-based partition control.

Notes:
- Hints affect placement/prefetch only; data and ordering remain unchanged.
- If a feature is disabled in `CAP.XMEM.CAP`, related hints are RAZ/WI and must not trap.

### 6.3 Swizzle operations (`XSWZ.*`)
| Mnemonic                                                | Summary                              |
|---------------------------------------------------------|--------------------------------------|
| **XSWZ.TEX2TEN dst, src, layout_tex, layout_ten, tile** | Texture -> tensor layout conversion. |
| **XSWZ.TEN2TEX dst, src, layout_ten, layout_tex, tile** | Tensor -> texture layout conversion. |

Notes:
- Layout identifiers are implementation-defined unless standardized; operations must preserve data values.
- Swizzle ops are hints for layout conversion; correctness is invariant to internal realization.

### 6.4 Optional inline compression

| Mnemonic                               | Summary                |
|----------------------------------------|------------------------|
| **XMEM.DLOADC rd, [addr], len, mode**  | Load compressed data.  |
| **XMEM.DSTOREC [addr], rs, len, mode** | Store compressed data. |

### 6.5 Compression I/O alignment

- Defaults (`mode/level/dictionary`) come from `CAP.XMEM.COMP_CTL`.
- If `CAP.XMEM.CAP.COMPRESS=0`, `XMEM.DLOADC/DSTOREC` must trap as illegal or act as normal LD/ST (implementation choice, documented).
- NOTE: If a domain lacks compression support, it must either trap or execute uncompressed while setting `COMP_ERR` in `CAP.XMEM.EVENTS` (Section 11).

---
## 7. Ordering, Fences & Scopes

This section describes ordering primitives and how scope/domain/class selections affect visibility. Scopes follow Section 3.1; domains/classes come from CAP or descriptor overrides.

### 7.1 Scope and Domain Model

- Fence scopes use `THREAD < WARP < WG < CLUSTER < SYSTEM`.
- Default domain is `CAP.XMEM.DOM` unless overridden; class mapping from `CAP.XMEM.CLSMAP` or descriptor `mclass`.
- Fences order effects with respect to scope and domain at execution; mid-instruction CSR writes do not affect in-flight ops.

### 7.2 Fence and Ordering Operations

- Cross-domain ordering: `XFENCE.PIPE {rt|mtx|all}` orders writes below the specified domain for subsequent reads.
- Event tokens: `XFENCE.SIGNAL` / `XFENCE.WAIT` synchronize producers/consumers; visibility still follows scope/domain rules.
- Cache/LDS visibility: producers writing LDS must execute `XLDS.BAR >=CLUSTER` or `XMEM.CCTL FLUSH` before signaling consumers reading via L1/L2.
- Global fence: if `ff=global`, completion includes DMA/streams whose policy came from current CAP defaults or a descriptor override.
- CLASS_PART: when `CLASS_PART=1`, fences must honor the active class selection (descriptor `mclass` if non-zero, else `CAP.XMEM.L1CFG.ACTIVE_CLASS`).

### 7.3 Ordering Rules and Guarantees

- Fences/barriers order all memory ops issued by the engine up to the specified scope, including LDS ops, XMEM loads/stores, and cache controls.
- `XMEM.PREF` is non-faulting and provides no visibility guarantee; ordering requires explicit fence/barrier.
- Streaming/DMA are included in ordering only when a global fence is requested (`ff=global`).
- Locks/pins via `XMEM.CCTL` (`LOCK/UNLOCK`) follow the same scope ordering; correctness must not depend on internal lock implementation.

### 7.4 Notes for Implementers (Informative)

- Preserve shared meaning of scopes across engines; avoid engine-local reinterpretation.
- Ensure mirrors (if present) are synchronized before fences issue so ordering reflects effective CAP state.
- Document coverage of `ff=global` for DMA/stream engines; software must not assume beyond documented coverage.

---

## 8. Security & Isolation

Optional security partitioning is enabled by `SEC_DOM`; when disabled, related fields are RAZ/WI.

### 8.1 Security Tagging

- When `SEC_DOM=1`, LDS allocations, cache ways, and DMA/stream contexts may carry `sec` (recommended 2 bits): `00=public`, `01=trusted`, `10=secure`, `11=reserved`.
- Security tags may come from CAP defaults or descriptor overrides; if absent, the default is implementation-defined and must be documented.

### 8.2 Enforcement and Events

- Violations of active security policy must set `SEC_VIOL` in `CAP.XMEM.EVENTS` / `XMIR.EVENTS`.
- Enforcement applies across LDS, cache classes, and DMA/stream engines using XMEM policy.
- If `SEC_DOM=0`, security tags are ignored (RAZ/WI); accesses must not fault based solely on `sec`.

### 8.3 Interaction with Domains and Classes

- Security tags are orthogonal to `dom` and `mclass`; accesses must satisfy both domain/class rules and security policy.
- Partitioning (`CLASS_PART`) and tiers do not alter security semantics; they affect placement only.

### 8.4 Notes for Implementers (Informative)

- Document supported `sec` encodings and any defaults.
- Ensure mirrors reflect security-related state; report violations uniformly via EVENTS.
- TODO: Clarify interaction with other XPHMG security mechanisms if added in future revisions.

---

## 9. Error Model & Events

This section describes architecturally visible error reporting and non-architectural counters.

### 9.1 Architectural Events (W1C)

`CAP.XMEM.EVENTS` (and `XMIR.EVENTS` if implemented) provide sticky W1C bits:
- `INV_MISS`   - invalid address or miss beyond bounds (where applicable)
- `LOCK_OVF`   - cache lock overflow/illegal lock
- `LDS_OOM`    - LDS allocation failure/out-of-memory
- `DMA_ERR`    - DMA/stream engine error related to XMEM policy
- `COMP_ERR`   - compression/decompression error or unsupported mode
- `BUDGET_OVF` - LDS budget exceeded (best effort)
- `SEC_VIOL`   - security tag violation (Section 8)
- `HYB_FALLBK` - hybrid class fallback invoked

Notes:
- Events are sticky until written with 1 (W1C); reads do not clear bits.
- If a feature is disabled in `CAP.XMEM.CAP`, related event bits are RAZ/WI.
- Event signaling must be consistent across engines; mirrors (if present) must match CAP.EVENTS at instruction boundaries.

### 9.2 Faults vs. Events

- Faults (illegal instruction, access fault, etc.) follow base ISA/engine rules; XMEM events are status only and do not mandate traps.
- If an operation traps (e.g., compressed ops when `COMPRESS=0` and implementation chooses to trap), event bits may still be set for diagnostics but do not replace trap behavior.

### 9.3 Non-Architectural Counters (Informative)

Counters such as `stall_mem`, `dma_bytes`, `lds_hits`, `l1_hits`, `l2_hits`, `streams_armed`, `lines_locked`, `cycles_busy_rt`, `cycles_busy_mtx` are informative only. If exposed:
- Document as PMU/debug registers outside CAP.
- Do not rely on them for architectural correctness.
- Ensure counters do not interfere with CAP.XMEM state or ordering.

---

## 10. Reset & Initialization Behavior

This section describes architectural reset behavior for XMEM CSRs. Unless stated, `CAP.XMEM.*` reset to 0 and mirrors (if present) must reflect the reset CAP state.

### 10.1 Reset Defaults (CAP-owned)

| CSR                     | Default                                 |
| ----------------------- | --------------------------------------- |
| `xmem_l1cfg.ACTIVE_CLS` | 0 (single class)                        |
| `xmem_clsmap`           | CL0=all ways, others=0                  |
| `xmem_descpol`          | DEF_CLASS=0, HYB_ENABLE=1, STRICT_DOM=0 |
| `xmem_lds_budget`       | 0 (unlimited)                           |
| All new CSRs            | Read as 0 if unimplemented              |

Notes:
- On reset, XMEM behaves as if CAP memory registers are zero unless documented otherwise; software must not assume non-zero defaults beyond the table.
- Feature gating via `CAP.XMEM.CAP` applies after reset; tied fields are RAZ/WI.
- If mirrors exist, synchronize them to CAP at or before the first instruction boundary after reset.

### 10.2 Initialization Considerations (Informative)

- Program `CAP.XMEM.CLSMAP`, `L1CFG`, `L2CFG`, `DOM`, streaming/compression defaults early in boot; do not rely on zeroed defaults.
- Set descriptor policy (`DESCPOL`) and budgets before enabling workloads that depend on partitioning or hybrid flows.
- TODO: Clarify platform-level boot flows that pre-initialize CAP.XMEM fields if applicable.

---
## 11. Compatibility & Interoperability

This section defines how XMEM remains consistent across domains and interoperates with other XPHMG components.

### 11.1 Cross-Domain Interoperability & Implementation Requirements (Normative)

All domains (RSV, MTX, RT, GFX, NPU, scalar ALU) must appear to execute under a unified XMEM model defined by CAP.XMEM.

#### 11.1.1 Architectural Source of Truth

- `CAP.XMEM.*` is authoritative for classes, domains, LDS budgets, streaming defaults/profiles, compression defaults, descriptor policies, and SVMEM configuration.
- Mirrors are permitted but must be synchronized at instruction boundaries and must not diverge architecturally.

#### 11.1.2 Instruction-Boundary Latching

- At each instruction boundary, engines latch effective `CAP.XMEM.*`: `L1CFG/L2CFG`, `CLSMAP/LDS_BUDGET`, `DOM`, `STREAM_CTL/SPROF*`, `COMP_CTL`, `DESCPOL`, `SVM*`.
- Mid-instruction CSR writes do not affect in-progress operations.

#### 11.1.3 SVMEM Consistency

- SVMEM fields: `SVMBASE`, `SVMIDX`, `SVMSCL`, `SVMLEN`, `SVMMODE`.
- Predication: masked lanes must not issue memory accesses.
- ZMODE: masked lanes follow `CAP.PREC.MODE.ZMODE` (merge vs zeroing).
- FOF: fault-only-first stops on the same element index for all domains.
- Element width: if `SVMLEN=0`, derive from `CAP.PREC.STAT.EFF_EW`.

#### 11.1.4 Streaming Profiles & Prefetch

- Defaults from `CAP.XMEM.STREAM_CTL`; descriptor `sprof_id` may override when non-zero.
- Prefetch depth/stride and non-temporal policy must not diverge across domains for the same profile.

#### 11.1.5 Memory Classes & Domains

- `CAP.XMEM.DOM` defines default coherence behavior; `CAP.XMEM.CLSMAP` maps CL0-CL3 to ways/QoS.
- `CAP.XMEM.LDS_BUDGET` provides soft class budgets.
- Domains must not redefine class/domain mappings.

#### 11.1.6 LDS Behavior

- `XLDS.ALLOC/FREE` and `XLDS.BAR.*` apply uniformly across domains.
- LDS alignment/layout rules must not vary per domain.

#### 11.1.7 Descriptor Integration (`XPHMG_MEMCTL`)

- Descriptor fields override CAP defaults only when set: `dom`, `mclass`, `sprof_id`, `hyb`, `sec`.
- Unspecified fields fall back to `CAP.XMEM.DESCPOL` or other CAP defaults.

#### 11.1.8 Compression Consistency

- When `CAP.XMEM.COMP_CTL` enables compression, all domains use the same default mode/dictionary.
- If a domain lacks compression support, it must ignore compressed hints and issue uncompressed accesses while setting `COMP_ERR` in `CAP.XMEM.EVENTS`.

#### 11.1.9 Error & Event Reporting

- All domains report to `CAP.XMEM.EVENTS` for OOB/misaligned accesses, LDS exhaustion, lock overflows, compression failures, security violations, and hybrid fallback.

#### 11.1.10 Deterministic Replay / Debug

- Debuggers must observe latched XMEM state per instruction boundary.
- Trace tools should present XMEM/SVMEM state synchronized with CAP.PREC state.
- Engines must not expose partially updated XMEM configurations.

#### 11.1.11 Summary (Normative)

- Determinism: no mid-instruction effects.
- Cross-domain consistency: unified SVMEM, ZMODE, FOF, streaming, LDS, class mapping.
- Descriptor coherence: per-job overrides limited to documented fields.
- Correctness: deviations must raise a memory-event sticky.
- Portability: pipelines (RT->MTX->RSV->GFX->NPU) remain valid across implementations.

### 11.2 Compatibility

- Precision and numeric control are delegated to `XPHMG_PREC`.
- Features missing from `xmem_cap` may be ignored without functional impact.
- Legacy `XMEM.DMA` / `XMEM.FENCE` opcodes are deprecated/reserved.
- Existing opcodes (`XLDS.*`, `XMEM.LDG/STG/PREF/CCTL`, `XSWZ.*`, `XMEM.DLOADC/DSTOREC`) remain; configuration source is CAP.

---

## 12. Programming Considerations / Profiles (Informative)

Advisory profiles and configuration patterns; no new requirements.

### 12.1 Example Profiles

| Use Case             | Recommended Setting                                              |
| -------------------- | ---------------------------------------------------------------- |
| NPU-only             | `CLASS_PART=0`, single CL0.                                      |
| RTU-only             | `CLASS_PART=0`, single CL0.                                      |
| Hybrid (NPU+RTU)     | `CLASS_PART=1`, `ACTIVE_CLS=2` or `3`. CL0=RT, CL1=MTX, CL2=HYB. |
| Unified              | Same as Hybrid; scheduler merges queues under CL2.               |

NOTE: Advisory only; normative rules are in Sections 3-11.

### 12.2 Configuration Guidelines (Informative)

- Program `CAP.XMEM.CLSMAP`, `L1CFG`, `L2CFG`, `DOM`, and streaming/compression defaults during initialization; do not rely on reset values.
- Use descriptor overrides (`mclass`, `sprof_id`, `dom`, `hyb`, `sec`) only when needed; defaults come from CAP.XMEM.
- Select fence scopes (`XLDS.BAR`, `XFENCE.*`) consistent with intended visibility; use `ff=global` when DMA/streams must be ordered.

### 12.3 Notes for Implementers (Informative)

- Profiles are illustrative only; conformance is defined by Sections 3-11.
- Document any platform-specific pre-configuration of CAP.XMEM (e.g., boot ROM settings) to avoid incorrect assumptions.
- TODO: Add further domain/class recipes if future revisions add classes or domain IDs.

---
## 13. Data Layout Model (XDL) (Informative)

Defines vocabulary and model for data layout in XMEM regions. Tiers describe placement/latency; XDL describes organization. XDL is informative and introduces no new opcodes.

### 13.1 Goals and Scope

- Provide shared concepts for MTX, GFX, RT, RSV, ACCEL layouts.
- Enable portable reasoning about tiling, alignment, swizzles.
- Avoid overspecifying hardware; encodings remain vendor-defined unless stated.

Consumed via:
- `XPHMG_MEMCTL` metadata (e.g., `mclass`, optional layout IDs)
- XMEM/XSWZ controls
- Engine ISA descriptions referencing XDL layouts

### 13.2 Core XDL Concepts

- **Element**: smallest scalar unit (e.g., FP16, BF16, FP8, INT8, INT32).
- **Lane**: contiguous vector of elements.
- **Tile**: 2D block `MxN`, used by MTX and tiled GFX.
- **Tensor**: N-D array (N >= 1), dimensions N,C,H,W or generic D0..Dn.
- **Surface**: 2D/2.5D GPU-like resource (render targets, textures).
- **Node/Record**: structured records (e.g., BVH nodes, triangles).

Engines may interpret a region differently based on `mclass` and mode.

### 13.3 Alignment and Granularity

Minimum guarantees:
- Minimal alignment sufficient for one tile/record (implementation-defined, typically >= 16 bytes).
- Elements forming a Lane/Tile/Tensor slice/Node are not split across non-contiguous spaces.

Further alignment may be defined per `mclass` (e.g., 32/64-byte BVH nodes, cacheline-aligned MTX tiles). NOTE: Alignment beyond the minimum is implementation-defined and must be documented; software should assume only the stated minimum unless a specific `mclass` requires more.

### 13.4 Canonical XDL Tile Layout (`XDL_TILE4x4`)

Mandatory canonical tile shared across MTX/RSV/GFX/RT when operating on tiles via XMEM:

**`XDL_TILE4x4_ROWMAJOR`** - 4x4 elements in row-major order:
```
t[0][0], t[0][1], t[0][2], t[0][3],
t[1][0], t[1][1], t[1][2], t[1][3],
t[2][0], t[2][1], t[2][2], t[2][3],
t[3][0], t[3][1], t[3][2], t[3][3]
```
Properties:
- Mandatory for all XPHMG implementations.
- Tiles must be element-contiguous and stride-aligned (>= 16 bytes; higher alignment may be required).
- Engines may reinterpret in row/column/flat views.
- All Tier-0 handoffs must use this layout unless `mclass` or metadata states otherwise.

### 13.5 Composite Tiles (`Linked Tiles`)

Larger tiles constructed from multiple 4x4 tiles placed contiguously.
```
XDL_TILE(M,N) = linkage of (M/4) x (N/4) canonical 4x4 tiles in row-major tile order.
```
Example: `XDL_TILE4x8` = two adjacent 4x4 tiles; `XDL_TILE8x8` = four 4x4 tiles in 2x2.
Rules:
- Engines may issue multiple ops on base tiles or fuse internally; architectural result must match the composite definition.
- Correctness is independent of internal fusion/scheduling. NOTE: Composite tiles are defined by logical concatenation; internal fusion does not change architectural layout/order.

### 13.6 Layout Identification (Conceptual)

Layout metadata may be associated implicitly via `mclass` or explicitly via `layout_id` (vendor-defined unless standardized). Examples: scalar/linear, SoA/AoS/AoSoA, tiled (4x4, 8x8), swizzled (Morton, block-compressed), tensor layouts (NCHW, NHWC, planar, interleaved). XDL standardizes names/semantics; numeric encodings are implementation-defined unless later standardized.

### 13.7 Relationship Between XDL and Tiers

- XDL is tier-agnostic: a layout can be used in Tier-0, Tier-1, or higher.
- Typical usage: Tier-0 for well-formed tiles/records; Tier-1 for constructing/transforming XDL entities; Tier-2+ for long-term storage/streaming.
- Engines should document supported XDL categories for performance. NOTE: When moving data across tiers/domains, layout metadata must remain consistent; engines must not reinterpret declared layouts.

### 13.8 Future Standardization

Future revisions may define standardized `layout_id` values for common cases and optional XMEM/XSWZ helpers for layout transforms. Until then, XDL serves as common language across XPHMG extensions without hardcoding vendor layouts.

---

## 14. Examples (Informative Appendix)

Examples illustrate how CAP.XMEM state and descriptor overrides drive behavior across domains. They are informative only.

### 14.1  RTU -> MTX Denoiser Chain (using CAP state)
```asm
# Configure CAP.XMEM.CLSMAP: CL0=RT (3 ways), CL1=MTX (5 ways)
li   t0, (0b00000111) | (0b11100000 << 8)
csrw CAP.XMEM.CLSMAP, t0

# Optionally update XMIR mirror if aliased
# csrw XMIR.CLSMAP, t0

# --- RT job ---
li   t1, DOM_RT
csrw CAP.XMEM.DOM, t1
memctl.dom    = DOM_RT
memctl.mclass = 0
xsubmit.rt t0, &desc_rt

xfence.signal 1

# --- MTX job ---
xfence.wait 1
li   t1, DOM_MTX
csrw CAP.XMEM.DOM, t1
memctl.dom    = DOM_MTX
memctl.mclass = 1
xsubmit.mtx t1, &desc_mtx
```

### 14.2  Unified Hybrid Task (Shared LDS)
```asm
# Hybrid / Unified domain
li   t0, DOM_ALL
csrw CAP.XMEM.DOM, t0

# Enable Hybrid sharing through CAP defaults
li   t1, ((1<<2) | (1<<0))  # HYB_ENABLE=1, DEF_CLASS=0
csrw CAP.XMEM.DESCPOL, t1

memctl.dom    = DOM_ALL
memctl.mclass = 2     # CL2 = HYB
memctl.hyb    = 1
xsubmit.mtx t0, &desc_unified
```

> Implementation note: If `XMIR.*` exists and operates in aliased mirror mode, software may program the same values through `XMIR.*`; both interfaces are equivalent at decode time. Mirrors must match CAP at instruction boundaries.

---

## 15. Normative Statement Summary

Key normative statements consolidated for reference:

- **N-1 Instruction-boundary latching:** Engines must observe `CAP.XMEM.*` at instruction boundaries; mirrors (if any) must match CAP at that boundary.
- **N-2 Capability gating:** If a `CAP.XMEM.CAP` bit is clear, requests targeting that feature are ignored; CSRs read as zero and ignore writes (RAZ/WI).
- **N-3 SVMEM predication/FOF:** Gather/scatter must honor RSV predication and `CAP.PREC.MODE.ZMODE`; with `FOF=1`, write first faulting element index to `RSV.SVFAULTI` and abort the operation.
- **N-4 LDS budgets/classes:** Enforce CAP budgets and classes on a best-effort basis; report overflows via `CAP.XMEM.EVENTS`.
- **N-5 Descriptor precedence:** Descriptor fields override CAP defaults only when non-zero and supported; otherwise CAP defaults apply.
- **N-6 Cross-domain consistency:** All domains must use the same meanings for `dom`, `mclass`, streaming profiles, compression defaults, and SVMEM semantics defined by CAP.
- **N-7 Ordering scopes:** Fence/barrier scopes follow Section 3.1; ordering is with respect to current `dom` and class selection.
- **N-8 Security (optional):** If `SEC_DOM=1`, security violations must raise `SEC_VIOL`; if `SEC_DOM=0`, security tags are RAZ/WI and must not fault.
- **N-9 Compression fallback:** If a domain lacks inline compression support, it must ignore compressed hints and issue uncompressed accesses while setting `COMP_ERR` in `CAP.XMEM.EVENTS`.

---
Designed for integration with `XPHMG_RSV`, `XPHMG_MTX`, `XPHMG_RT`, and `XPHMG_CAP`.

*Licensed under CC-BY-SA 4.0 - Open Specification maintained by Pedro H. M. Garcia.*
