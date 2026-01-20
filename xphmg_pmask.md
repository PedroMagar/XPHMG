# RISC-V Vendor Extension **XPHMG_PMASK**
**Predicate State, Mask Algebra, Banked Masks, and Predicate-Controlled Branch/Select**

**Category:** Vendor Extension (`XPHMG_PMASK`)  
**Version:** 0.1.1  
**Author:** Pedro H. M. Garcia  
**License:** CC-BY-SA 4.0 (open specification)

---

## 1. Overview
This section summarizes the scope of `XPHMG_PMASK`, identifies the
architectural state it defines, and explains how predicate state participates
in the programmer-visible contract across XPHMG domains. This overview is
informative except where it restates normative invariants defined later in this
specification.

`XPHMG_PMASK` defines a **unified architectural predicate model** used across XPHMG domains:
- Scalar core (control-flow, select),
- RT (hit/miss predicate, PRED_ONLY flows),
- XMEM (predicated loads/stores; masked lanes must not access memory),
- MTX/GFX future use (masked writeback, compaction decisions).

The model includes a scalar predicate (`P0`) and optional lane predicate masks
(`PMASK`) with an explicit bank-selection mechanism. Implementations MAY provide
only the scalar view, but when lane masks are present the ISA defines how scalar
and lane predication are interpreted. Predicate state is architecturally visible
to software and is consumed by control-flow and select operations; additional
domain uses are defined by their respective specifications and constrained by
the rules in this extension.

This extension standardizes branch predicates and mask algebra as architectural
state without requiring dependence on any other domain. It also
defines banked masks to reuse common masks. It does not mandate any particular
microarchitectural storage or implementation strategy for predicate state.
Software-visible behavior is limited to the architectural predicate state and
the effects of instructions defined herein; any additional internal handling is
microarchitecturally unspecified unless mandated by cross-domain rules.

This extension does **not** define numeric formats or rounding; those are
defined by `XPHMG_CAP`. Interactions with CAP and XMEM are normative where
referenced later; this overview does not introduce constraints beyond those
defined in the dependency and rules sections.

---

## 2. Dependencies and Interactions
This section defines architectural dependencies and cross-domain interactions
required to interpret predicate state consistently across XPHMG domains.
`XPHMG_PMASK` defines predicate state and predicate-manipulation semantics but
does not define numeric formats, rounding, or memory system behavior. A domain
that consumes predicate state MUST apply the rules in this extension and any
additional domain-specific rules defined by that domain. Behavior not explicitly
constrained here is defined by the consuming domain.

### 2.1 CAP numeric policy (mandatory)
`XPHMG_PMASK` relies on `XPHMG_CAP` to define the architectural policy for
masked-off writeback. The predicate state determines whether a lane or scalar
destination is written, and `CAP.PREC.MODE.ZMODE` determines the value written
to masked-off destinations. This section states the policy; detailed CAP
mechanisms and configuration rules are specified in the CAP document.

Predicated writeback across domains MUST follow `CAP.PREC.MODE.ZMODE`:
- `ZMODE=0`: masked-off lanes **merge** (preserve destination)
- `ZMODE=1`: masked-off lanes **zero** (write architectural zero)
No alternative merge/zero semantics are introduced by this extension.

### 2.2 XMEM interoperability (mandatory if XMEM predication is present)
This subsection defines the architectural contract between predicate state and
predicated memory operations. When XMEM predication is implemented, the active
mask bank selected by the executing instruction governs which lanes may issue
memory accesses. This extension does not define the memory model itself, but it
requires that masked-off lanes be architecturally invisible to memory.

Memory operations that honor masks MUST NOT issue memory accesses for masked-off
lanes. Address generation, speculation, or internal evaluation for masked-off
lanes is microarchitecturally unspecified provided it does not create
architectural memory side effects.
Mask selection must follow this specification's bank rules (see Section 3.2,
Section 3.3, and Section 4.3).

Predicate-false lanes are architecturally inactive: they MUST NOT perform
memory accesses, MUST NOT raise exceptions/faults, and MUST NOT contribute to
any first-fault lane selection. Any instruction that performs a memory access
under predication MUST evaluate predication before issuing the access, such
that masked-off lanes are suppressed at the point of access. If a domain
defines a first-fault lane index, it is determined only among predicate-true
lanes in increasing lane order. If all lanes are predicate-false, no access
occurs, no fault occurs, and the instruction completes with no side effects
(subject to CAP ZMODE writeback rules if applicable).

---

## 3. Architectural Predicate State (Programmer's Model)
This section defines the programmer-visible predicate state and the conceptual
model used by `XPHMG_PMASK`. It specifies the architectural elements that hold
predicate information, how they are interpreted by consuming domains, and which
properties are guaranteed by the ISA. The intent is to give software and
implementations a consistent predicate contract independent of any specific
domain, while deferring domain-specific usage details to the relevant
specifications.

`XPHMG_PMASK` defines a two-level predicate model:

### 3.1 Scalar predicate (`P0`)
A single 1-bit predicate used for scalar control flow and scalar select:
- `P0=1`: "true"
- `P0=0`: "false"

`P0` is consumed by `BP.*` when no lane mask is in use and by other scalar
predicate-controlled instructions defined by this extension. When lane masks
are implemented, `P0` is the scalar view of predicate state as defined by each
instruction's semantics (e.g., instructions may map `P0` to bit 0 of the
selected mask bank). When only scalar predication is implemented, `P0` exists
as a standalone architectural bit. The ISA does not require any additional
storage or shadow state beyond what is needed to maintain this architectural
bit.

### 3.2 Lane predicate masks (`PMASK`) (optional)
A bitmask representing active lanes (up to 64 bits for baseline; >64 is implementation-defined).

This subsection defines the vector-lane view of predicate state. Each bit of
`PMASK` corresponds to a lane in the consuming domain's effective vector length.
The mapping of lanes to bits is architectural and MUST be applied consistently
by all domains that honor masks. `XPHMG_PMASK` does not define how vector length
is configured or managed; the effective length and lane interpretation are
defined by the consuming domain.

Bank selection uses a 2-bit `pbank` field selecting `{0,1,2,3}`. `pbank=0`
selects the virtual ALL_ONES bank; `pbank=1..3` select programmable banks only
if implemented by the current profile/implementation. Selecting an
unimplemented programmable bank MUST raise Illegal Instruction (see Section 9).

Effective predication is defined as:

`pred[i] = PMASK.bank[pbank_eff][i]`

where `pbank_eff` is selected by instruction encoding, and `pbank=0` denotes a virtual ALL_ONES bank.

`PMASK` exists as architectural state even when only scalar predication is used
and may be implemented minimally as 1-bit (bit0 only). Software that requires
lane masks beyond bit 0 MUST target a conformance level that includes mask
banks and mask load/store support (see Section 7).
TODO: Clarify the required software-visible behavior when only the scalar
minimum is implemented and a non-minimum instruction encoding selects a mask
bank.
Implementations MAY implement 1-4 banks; `bank0` always exists and is virtual.
Any instruction encoding that selects an unimplemented programmable bank MUST
raise Illegal Instruction.

### 3.3 Banked mask model (new)
To reduce overhead in hot paths that repeatedly reuse common masks (e.g.,
ALL_ONES, tail/edge masks, "alive lanes" masks), `XPHMG_PMASK` defines **banked
predicate masks**.

The banked model provides multiple named mask banks that can be selected by
instructions using the `pbank` selector. This allows software to keep frequently
used masks resident in architectural state and select them without explicit
copying. The banked model does not change the predicate semantics; it only
defines how predicate state is organized and selected.

#### 3.3.1 Bank count
`pbank` is a 2-bit selector encoding `{0,1,2,3}`.
Baseline recommends **4 banks** addressed by `pbank`:
- `bank0` (pbank=0): **virtual ALL_ONES**
- `bank1` (pbank=1): primary programmable mask
- `bank2` (pbank=2): programmable
- `bank3` (pbank=3): programmable

Implementations MAY provide fewer programmable banks; `bank0` MUST always
exist. Banks `1..3` are programmable only if implemented by the current
profile/implementation. The number of programmable banks is an architectural
property that software can discover via the conformance and discovery
mechanisms defined in Section 7. Selecting an unimplemented programmable bank
MUST raise Illegal Instruction (see Section 9).

#### 3.3.2 bank0 is virtual ALL_ONES (normative)
`PMASK.bank0` is **virtual**:
- reads as `all_ones_for_effective_VL`
- writes are ignored
- `P0` view of bank0 is always 1

This provides a **free default predicate** for the common case.

Selecting `pbank=0` MUST yield a mask in which all lanes within the effective
vector length are active, independent of any prior predicate-manipulation
instructions. This applies to both mask-based operations and scalar views
(`P0`) that map to the selected bank.

#### 3.3.3 Programmable banks (bank1..bank3)
`bank1..bank3` are writable/readable (subject to implementation limits).

Programmable banks hold architecturally visible mask state that can be read and
modified by the predicate state and mask algebra instructions defined in this
extension. The width of each bank and the number of supported banks are
implementation-defined within the limits described above. Access to these
banks is architectural; their microarchitectural representation is unspecified.
Any instruction that selects an unimplemented programmable bank MUST raise
Illegal Instruction.

### 3.4 Writeback policy for masked-off lanes
This subsection describes how predicate state interacts with destination
writeback. Predicate evaluation determines whether a lane is active, while the
policy for masked-off lanes is defined by `XPHMG_CAP`. `XPHMG_PMASK` does not
introduce an independent policy, and it does not override or reinterpret the
CAP definition.

All operations that write vector/scalar destinations under a predicate MUST
obey `ZMODE`:
- Merge vs Zero behavior is controlled exclusively by CAP (or by a one-shot
  override mechanism if defined elsewhere).

NOTE: This section does not define any one-shot override mechanism; if such a
mechanism exists, it MUST be specified by the owning extension and remain
consistent with the CAP policy.

---

## 4. Instruction Set
This section defines the instruction semantics and operand conventions for
`XPHMG_PMASK`. It describes how predicate state is selected, updated, and
consumed by instructions, and it specifies the architectural results that are
visible to software. Encodings are described at a conceptual level; exact field
assignments are determined by the ISA map and are outside the scope of this
document.

### 4.1 Encoding space
This subsection establishes the recommended opcode space for `XPHMG_PMASK`
instructions. It does not define a finalized funct3/funct7 map, but it
identifies the placement intended for compatibility with other XPHMG
extensions.
This specification recommends placing `XPHMG_PMASK` ops under **custom-1 (`0101011`)**
alongside other XPHMG "OP" style instructions.

> NOTE: This file specifies semantics and mnemonics; exact funct3/funct7 allocation is finalized in the ISA map.

### 4.2 Operand conventions
This subsection defines operand and state conventions shared by all
`XPHMG_PMASK` instructions. `PMASK` is architectural state, so it does not
appear as an explicit register field, and the standard integer register fields
(`rd`, `rs1`, `rs2`) follow XLEN conventions unless noted otherwise.
- `rd/rs1/rs2` are XLEN registers unless explicitly noted.
- `PMASK` is implicit architectural state (no explicit register field).
- Some ops treat `P0` as a scalar view of the predicate:
  - `P0` maps to `PMASK[0]` when masks are present, else exists as a standalone flip-flop.

### 4.3 Bank selection (`pbank`)
This subsection defines the bank-selection mechanism used by instructions in
this extension. Bank selection is an architectural input to predicate
evaluation and predicate state updates; it does not introduce additional
register operands.
Many instructions in this extension take an implicit or explicit **`pbank`** selector:
- `pbank` is a **2-bit** immediate field (0..3) in the instruction encoding (recommended),
  or in an attached prefix when the encoding defines one.
- If an instruction does not encode `pbank`, it defaults to `pbank=0` (bank0, ALL_ONES).
`pbank=0` selects the virtual ALL_ONES bank; `pbank=1..3` select programmable
banks only if implemented. Any instruction encoding that selects an
unimplemented programmable bank MUST raise Illegal Instruction.

**Design goal:** avoid "select mask" helper instructions in hotpaths.

### 4.4 Predicate state ops (P0 / PMASK management)
This subsection defines instructions that initialize, move, or explicitly
update predicate state. These operations are the architectural mechanism for
loading masks, clearing masks, and transferring scalar predicate values.

### 4.4.1 Set / Clear (acts on a selected bank unless stated)
These instructions set or clear predicate state for a selected bank, and they
update `P0` as specified by their semantics. When `pbank=0`, the bank is
virtual and writes have no effect.
- `pmclr pbank`  
  **Semantics:** if `pbank!=0`: `PMASK.bank[pbank] <- 0`; `P0 <- PMASK.bank[pbank][0]`  
  If `pbank==0` (bank0): no effect (bank0 is virtual all-ones).

- `pmset pbank`  
  **Semantics:** if `pbank!=0`: `PMASK.bank[pbank] <- all_ones_for_effective_VL`; `P0 <- 1`  
  If `pbank==0`: no effect (already all-ones).

- `pmmov.p0 rd`  
  **Semantics:** `rd <- P0 ? 1 : 0`

- `pmmov.to_p0 rs1`  
  **Semantics:** `P0 <- (rs1 != 0)`; if masks present, this updates `PMASK.bank[pbank][0]` only
  for encodings that explicitly specify a `pbank` (optional). Recommended: scalar-only (does not touch banks).

> Rationale: `P0` is a scalar condition carrier; banked masks are for wavefront control.

### 4.4.2 Load / Store mask low/high (programmable banks only)
These instructions provide architectural access to the low and high 32-bit
segments of a selected mask bank. They are defined only for programmable banks;
`pbank=0` reads as all-ones and ignores writes.
All loads/stores below act on the selected `pbank`.
If `pbank==0`: reads return `all_ones`; writes are ignored.

- `pmlow.rd  pbank, rd`  
  **Semantics:** `rd <- PMASK.bank[pbank][31:0]` (or all-ones for bank0)

- `pmhigh.rd pbank, rd`  
  **Semantics:** `rd <- PMASK.bank[pbank][63:32]` (or all-ones for bank0)

- `pmlow.wr  pbank, rs1`  
  **Semantics:** if `pbank!=0`: `PMASK.bank[pbank][31:0] <- rs1`

- `pmhigh.wr pbank, rs1`  
  **Semantics:** if `pbank!=0`: `PMASK.bank[pbank][63:32] <- rs1`

### 4.5 Mask algebra ops (boolean ops on PMASK)
This subsection defines boolean operations on the selected mask bank. These
operations modify predicate state in-place and optionally update the scalar
view (`P0`) as defined below.

All mask algebra operations:
- update `PMASK.bank[pbank]` (and set `P0 <- PMASK.bank[pbank][0]`), and
- reduce to scalar behavior if only `P0` exists.

### 4.5.1 Unary
The unary form applies a bitwise NOT to the selected mask bank when it is
programmable. The virtual bank (`pbank=0`) is not modified.
- `pmnot pbank`  
  **Semantics:** if `pbank!=0`: `PMASK.bank[pbank] <- ~PMASK.bank[pbank]`

### 4.5.2 Binary (PMASK <- op(PMASK, src))
The binary forms apply a boolean operation between the selected mask bank and a
source mask derived from `rs1`. The exact interpretation of `rs1` is defined by
the mask sourcing rules below.

- `pmand  pbank, rs1`  
  `PMASK <- PMASK & rs1_mask`
- `pmor   pbank, rs1`  
  `PMASK <- PMASK | rs1_mask`
- `pmxor  pbank, rs1`  
  `PMASK <- PMASK ^ rs1_mask`
- `pmandn pbank, rs1`  
  `PMASK <- PMASK & ~rs1_mask`

Where `rs1_mask` is taken as:
- if masks present: a 64-bit mask materialized from registers (convention-defined), or an immediate form (impl-defined);
- if only scalar P0: `rs1_mask = (rs1 != 0)`.

> Note: A future revision may add explicit two-register mask sources; this draft keeps encoding compact.

### 4.6 Predicate tests and generation
This subsection defines instructions that compute scalar results or predicate
state from masks or scalar operands. These operations do not modify mask banks
unless explicitly stated.

#### 4.6.1 Mask Tests / Reductions (write scalar results)
These instructions reduce the selected mask to a scalar result written to `rd`.
They do not modify any mask bank.

- `pmany  pbank, rd`  
  `rd <- (PMASK.bank[pbank] != 0) ? 1 : 0`

- `pmnone pbank, rd`  
  `rd <- (PMASK.bank[pbank] == 0) ? 1 : 0`

- `pmall  pbank, rd`  
  `rd <- (PMASK.bank[pbank] == all_ones_for_effective_VL) ? 1 : 0`

- `pmpopcnt pbank, rd`  
  `rd <- popcount(PMASK.bank[pbank][0..VL-1])`

#### 4.6.2 Compare/Test -> Predicate (hotpath support)
These ops produce predicate bits **without** materializing 0/-1 vectors.
They are intended for scalar hot paths that set predicate state directly.

##### 4.6.2.1 Scalar compare -> P0
This form compares two XLEN operands and writes the comparison result to `P0`.
It does not modify any mask bank unless otherwise stated by the instruction
definition.
- `pcmp.<cond>.<type> rs1, rs2`  
  **Semantics:** `P0 <- cond(rs1, rs2)`.

Supported `<cond>` (minimum):
- `eq, ne, lt, le, gt, ge`

Supported `<type>` (minimum):
- `.u` unsigned XLEN
- `.s` signed XLEN

##### 4.6.2.2 Test (a & b) != 0 -> predicate (vptest-like)
This form computes a bitwise AND and writes the nonzero test to `P0`.
- `ptestnz` scalar: `P0 <- ((rs1 & rs2) != 0)`

### 4.7 Predicate-controlled select (SETcc/CSEL equivalents)
This subsection defines a scalar select that chooses between two operands based
on the current predicate state.

### 4.7.1 Scalar select by P0
The scalar select reads `P0` and writes one of the two input operands to `rd`.
- `psel rd, rs1, rs2`  
  `rd <- (P0 ? rs1 : rs2)`

### 4.8 Predicate branches
This subsection defines predicate-controlled branches. The scalar form uses
`P0`, while optional bank-aware forms use the selected mask bank.

### 4.8.1 Scalar branch on P0
These branches are defined for scalar control flow and are evaluated using
`P0`. Assemblers may provide pseudonyms, but the architectural behavior is that
of a branch on the predicate bit.
- `bp.t  label`  (branch if P0==1)  
- `bp.f  label`  (branch if P0==0)

Assemblers may provide pseudonyms:
- `bep z, label`   (branch if predicate == 0)
- `bep nz, label`  (branch if predicate != 0)

### 4.8.2 Bank-aware branch (optional; fused hotpath form)
If masks are present, these fused forms MAY be provided:

- `bp.any  pbank, label`   (branch if any lane active in bank)
- `bp.none pbank, label`   (branch if no lane active in bank)

These are equivalent to `pmany/pmnone` followed by `bp.t`, but are provided as fused forms.

---

## 5. Architectural Rules and Determinism
This section defines global architectural rules that ensure predicate state is
updated deterministically and observed consistently across XPHMG domains. These
rules apply to all instructions that read or write predicate state, including
domain-specific instructions that consume `P0` or `PMASK`. The rules below are
normative and constrain architecturally visible behavior; microarchitectural
mechanisms are otherwise unspecified.

1) All domains that update predicate MUST do so deterministically at the instruction boundary.
2) Masked lanes MUST obey `CAP.PREC.MODE.ZMODE` (merge vs zero).
3) Memory ops that honor masks MUST NOT issue memory accesses for masked-off lanes.

Rule (1) requires that the architectural predicate state after each instruction
is uniquely defined by program order and the instruction's specified semantics.
Speculation or out-of-order execution is permitted only if it does not alter the
architecturally visible predicate state at instruction retirement.
NOTE: This specification does not define additional ordering guarantees between
different domains beyond instruction ordering.

Rule (2) restates the CAP dependency for clarity in the architectural rules
section. It applies uniformly to scalar and lane predication whenever a masked
writeback occurs, regardless of the domain performing the writeback.

Rule (3) defines the architectural visibility of predicated memory operations.
Any access that would have been suppressed by a masked-off lane MUST be
architecturally absent. Internal prefetching or speculation is allowed only if
it is not architecturally observable.
TODO: Clarify how faulting memory operations interact with masked-off lanes in a
future revision.

---

## 6. Examples
This section provides informative examples illustrating how predicate state
is used by instructions defined in this extension and by domain-specific
instructions that consume predicate state. The examples are intended to clarify
usage patterns, not to define additional semantics. Implementations must
conform to the normative rules in earlier sections regardless of the specific
coding style shown here.

### 6.1 RT hit test + branch on predicate (scalar view)
This example demonstrates a scalar predicate-producing operation in the RT
domain followed by a predicate branch. The intent is to show how a domain
instruction can set `P0` and how a subsequent branch consumes that predicate.
The branch pseudonym used by the assembler does not alter the architectural
effect of branching on `P0`.
```asm
RT.BBOX  imm=(1<<2) /*PACK_HINT*/, rs1=a0, rd=a1
bep  nz, %skip_hit   # assembler pseudonym: branch on predicate false
````

### 6.2 Using bank2 for "tail mask" and bank3 for "alive lanes"
This example shows how programmable banks can hold reusable masks. One bank
stores a tail mask and another stores an "alive lanes" mask. The code then
branches based on mask reductions without explicitly moving masks between
banks. The example assumes that programmable banks and bank-aware branches are
implemented (see Section 7 for conformance levels).

```asm
# Preload tail mask into bank2
pmlow.wr  pbank=2, t0
pmhigh.wr pbank=2, t1

# Preload alive mask into bank3
pmlow.wr  pbank=3, t2
pmhigh.wr pbank=3, t3

# Branch if any tail lanes active
bp.any pbank=2, %do_tail

# Early-out if none alive
bp.none pbank=3, %done
```

---

## 7. Conformance and Discovery
This section defines the conformance levels for `XPHMG_PMASK` and the
architectural intent for feature discovery. It is intended to allow software
to identify the available predicate functionality (scalar-only versus banked
masks) and to select compatible code paths. The specific discovery mechanism is
defined by CAP and is referenced here for consistency.
TODO: Specify discovery/feature-bit mapping in a future revision.
TODO: Standardize naming of conformance sets in a future revision.

### 7.1 Minimal Conformance Sets
This subsection enumerates the minimum instruction subsets required for a
conforming implementation. The sets are ordered from the smallest scalar-only
baseline to banked-mask support. The list is descriptive of architectural
capability and does not imply any additional encoding requirements beyond those
defined in Section 4.

* **PMASK-M1 (Minimum):**
  `pmclr, pmset, pmmov.p0, pmmov.to_p0, pmany, pmnone, psel, bp.t, bp.f`
  (bank0 MUST exist as virtual ALL_ONES)

* **PMASK-M2 (Banked masks):**
  adds `pmlow/pmhigh` reads/writes with `pbank`, boolean ops (`pmnot/pmand/pmor/pmxor/pmandn`),
  reductions (`pmpopcnt`) and bank-aware branches (`bp.any/bp.none`).

Implementations SHOULD advertise bank count and conformance level via CAP feature bits (TBD).
NOTE: The exact feature-bit layout and reporting mechanism are not defined in
this revision and must be specified by `XPHMG_CAP` or a future revision of this
extension.

---

## 8. Reset and Initialization (TBD)
This section defines the architectural reset and initialization behavior for
predicate state. It describes what software may assume about `P0` and `PMASK`
immediately after reset and during any architecturally defined initialization
sequence. In v0.1, the precise reset values for predicate state are not yet
specified.

NOTE: This behavior is intentionally left unspecified in v0.1 and may be clarified in a future revision.
TODO: Clarify whether `P0` is reset to a defined value or is implementation-defined in a future revision.
TODO: Clarify whether programmable mask banks are reset to zero, all-ones, or an implementation-defined value in a future revision.

## 9. Exceptions and Illegal Instructions (TBD)
This section describes the exception behavior associated with `XPHMG_PMASK`
instructions and predicate state usage. It defines how illegal encodings or
unsupported features are reported and clarifies which behaviors are
architecturally specified versus intentionally unspecified in this revision.

NOTE: This behavior is intentionally left unspecified in v0.1 and may be clarified in a future revision.
TODO: Clarify the required exception behavior when an implementation does not support a referenced `pbank`.
TODO: Clarify the required exception behavior for unimplemented instruction subsets beyond the advertised conformance level.
