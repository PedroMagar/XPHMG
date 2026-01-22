# **XPHMG - Experimental RISC-V Extension Family**

**XPHMG** is a family of **RISC-V vendor extensions** that defines a unified **architectural execution substrate** targeting graphics, geometry, and compute workloads.

Rather than exposing a fixed graphics pipeline, driver-managed heuristics, or a high-level graphics API, XPHMG provides **explicit architectural primitives** that software composes directly.
All observable behavior — numeric precision, memory residency, coherence, and execution ordering (at the architectural contract level) — is controlled through architecturally visible state.

This repository hosts the **XPHMG v0.1 specification**, the first stable architectural baseline.

---

## ✦ Design Philosophy

XPHMG is built around a small set of core principles:

* **No hidden heuristics**
  No implicit scheduling, memory promotion, or driver inference.

* **Explicit state**
  Numeric policy and memory behavior are first-class architectural state.

* **Modular extensions**
  XPHMG is a family of opt-in extensions linked by explicit architectural contracts, rather than a single monolithic execution model.

* **Deterministic and debuggable**
  Identical code and state must produce identical results.

* **Heterogeneous by design**
  Graphics, compute, ray traversal, and accelerators share the same substrate.

XPHMG is not a replacement ISA. It is designed to coexist with a standard RISC-V core, extending it with:

* Explicit predication and masking mechanisms
* A shared, predicable memory access model
* Centralized execution and masking policies
* A scalar–vector reinterpretation model for compact and explicit data-parallel execution
* Optional acceleration domains independent of graphics or GPU functionality

Each extension defines its own responsibilities and relies on common architectural contracts rather than duplicated semantics.

---

## ✦ What XPHMG Is (and Is Not)

### ✔ What it *is*

* A **low-level execution model** for graphics and compute on RISC-V
* A **set of interoperable extensions** sharing a common architectural contract
* A foundation for:
  * tile-based rendering,
  * visibility-driven pipelines,
  * hybrid graphics + ML workloads,
  * research and experimental hardware
* An explicit **compute execution model**, enabling:
  * data-parallel and conditional execution without fixed pipelines,
  * diverse execution styles via explicit predication and scalar–vector reinterpretation


### ✖ What it *is not*

* A graphics API (Vulkan, DirectX, OpenGL)
* A driver or runtime model
* A fixed-function graphics pipeline
* A replacement for RISC-V base or standard extensions

---

## ✦ Architectural Overview

XPHMG is organized as a **family of extensions**, each with a clearly defined role:

| Extension       | Role                                                               |
| --------------- | ------------------------------------------------------------------ |
| **XPHMG_CAP**   | Authoritative capability discovery and numeric/memory policy       |
| **XPHMG_PMASK** | Explicit execution masks and predication control                   |
| **XPHMG_XMEM**  | Explicit memory control (LDS, classes, domains, streaming, SVMEM)  |
| **XPHMG_RSV**   | Scalar–vector reinterpretation for compact data-parallel execution |
| **XPHMG_MTX**   | Small matrix and tile compute primitives                           |
| **XPHMG_RT**    | Ray and geometry intersection primitives                           |
| **XPHMG_GFX**   | Texture, interpolation, shading, and visibility helpers            |
| **XPHMG_ACCEL** | Descriptor-driven integration of NPUs and custom accelerators      |

All extensions **consume the same CAP and XMEM state**, ensuring consistent behavior across domains.

---

## ✦ Architectural Scope

### Base Substrate
The XPHMG base substrate defines the architectural foundation shared by all execution domains.
It establishes explicit policy state, predication mechanisms, and memory semantics through **CAP**, **PMASK**, and **XMEM**, ensuring consistent behavior across compute, graphics, and accelerator extensions.

### Compute Execution
The compute execution domain builds upon the base substrate to provide explicit data-parallel and conditional computation.
It is primarily expressed through the **RSV** execution model, with support from compact matrix primitives (**MTX**) and descriptor-driven accelerators (**ACCEL**), enabling flexible compute datapaths for irregular, DSP-style, and experimental workloads.

### Graphics & Geometry
The graphics and geometry domain builds upon the base substrate to support explicit visibility, texturing, interpolation, and traversal operations.
Through shared memory semantics and domain-controlled execution, XPHMG enables software-defined graphics and geometry pipelines, including tile-based and visibility-driven models, without imposing fixed-function stages or implicit scheduling.

XPHMG supports **tile-based rendering and tiled compute** through:

* Explicit LDS/XMEM allocation,
* Tiered memory residency (Tier-0 / Tier-1),
* Domain-controlled ownership and handoff,
* Canonical tile layouts shared across engines.

Tiles are **software-defined working sets**; their size, lifetime, and layout are fully controlled by software and are not architectural objects.

### Specialized Extensions
Specialized extensions extend the XPHMG substrate with domain-specific functionality without altering the core architectural contracts.
This includes descriptor-driven accelerator integration (ACCEL) and optional compute or signal-processing blocks, allowing implementations to incorporate custom or application-specific engines while preserving uniform execution and memory semantics.

---

## Key Architectural Decisions

The following architectural choices are central to the validity and coherence of the XPHMG design:

### Explicit Predication via PMASK

Predication is implemented through **explicit execution masks**, selected per instruction.
There is no implicit or global predicate state.

PMASK supports multiple banks, with **bank 0 hardwired to an all-active mask**, providing a canonical neutral state and allowing instructions to execute unmasked by default.

### Opt-In, Non-Global Semantics

Predication and masking are only applied where explicitly specified.
Instructions that do not reference PMASK are unaffected, preserving predictable scalar execution.

### CAP as a Central Policy Authority

Execution policies such as masked writeback behavior, merge vs. zero semantics, and format capabilities are centralized in **CAP**, avoiding semantic duplication across extensions.

### XMEM as an Explicit Memory Model

Memory behavior under predication is defined by **XMEM**, including:

* Masked-off accesses
* Exception and fault handling
* First-fault and lane-selective semantics

This model is shared across all consuming domains.

### Separation of Predicate Production and Consumption

Condition generation (e.g., comparisons, traversal results) is decoupled from mask application.
PMASK applies effects but does not encode control flow.

### Value Independent of Graphics or GPU Extensions

Extensions such as **RSV**, **XMEM**, **PMASK**, and **ACCEL** form a complete execution and acceleration model independent of graphics or ray tracing.

### Explicit Architectural Cost

Additional state, gating logic, and masking behavior are explicitly defined in the specification.
No extension assumes hidden or zero-cost behavior.

---

## ✦ Versioning & Stability

* **Current version:** **v0.1.1**
* **Status:** Stable architectural baseline

Version 0.1 defines:

* the CAP authority model,
* unified memory semantics,
* execution-domain composition rules,
* instruction-level contracts for all extensions.

Version 0.1.1 defines:

* PMASK as source for predicate and execution mask.


---

## ✦ Repository Contents

```
/
├─ xphmg.md                 # Root architecture overview
├─ xphmg_cap.md             # Capabilities & numeric/memory policy
├─ xphmg_pmask.md           # Predication & execution masks
├─ xphmg_xmem.md            # Explicit memory model
├─ xphmg_rsv.md             # Scalar-vector execution model
├─ xphmg_rsv-profiles.md    # Optional RSV instruction profiles
├─ xphmg_mtx.md             # Matrix & tile compute
├─ xphmg_rt.md              # Ray traversal & intersection
├─ xphmg_gfx.md             # Graphics & shading helpers
├─ xphmg_accel.md           # Descriptor-driven accelerators
```

These documents are normative and define the authoritative behavior of the architecture.
All documents are part of the **XPHMG v0.1 specification**.

---

## ✦ Intended Audience

XPHMG is intended for:

* Hardware architects and RTL designers
* Compiler and toolchain developers
* Graphics and compute researchers
* Experimental SoC and accelerator projects
* Anyone interested in explicit, low-level graphics/compute execution models

---

## ✦ Status

This repository represents the **public release** of XPHMG.

Feedback, discussion, and experimentation are welcome — especially around:

* execution-domain composition,
* memory semantics,
* tile-based and hybrid pipelines,
* accelerator integration.

XPHMG is under active architectural development.
The specification prioritizes correctness, explicitness, and composability over short-term stability.

---

## ✦ License

The XPHMG specification is released under **CC-BY-SA 4.0**.
