# **XPHMG — eXtensible Pipeline for Heterogeneous Media & Graphics**

**XPHMG** is a family of **RISC-V vendor extensions** that defines a unified **execution substrate** for modern graphics, geometry, and compute workloads.

Rather than exposing a fixed graphics pipeline, driver-managed heuristics, or a high-level graphics API, XPHMG provides **explicit architectural primitives** that software composes directly.
All observable behavior — numeric precision, memory residency, coherence, and execution ordering — is controlled through architecturally visible state.

This repository hosts the **XPHMG v0.1 specification**, the first stable architectural baseline.

---

## ✦ Design Philosophy

XPHMG is built around a small set of core principles:

* **No hidden heuristics**
  No implicit scheduling, memory promotion, or driver inference.

* **Explicit state**
  Numeric policy and memory behavior are first-class architectural state.

* **Composable, not prescriptive**
  XPHMG defines primitives and conventions, not pipelines or APIs.

* **Deterministic and debuggable**
  Identical code and state must produce identical results.

* **Heterogeneous by design**
  Graphics, compute, ray traversal, and accelerators share the same substrate.

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

### ✖ What it *is not*

* A graphics API (Vulkan, DirectX, OpenGL)
* A driver or runtime model
* A fixed-function graphics pipeline
* A replacement for RISC-V base or standard extensions

---

## ✦ Architectural Overview

XPHMG is organized as a **family of extensions**, each with a clearly defined role:

| Extension       | Role                                                              |
| --------------- | ----------------------------------------------------------------- |
| **XPHMG_CAP**   | Authoritative capability discovery and numeric/memory policy      |
| **XPHMG_XMEM**  | Explicit memory control (LDS, classes, domains, streaming, SVMEM) |
| **XPHMG_RSV**   | Scalar-vector reinterpretation for compact vectorization          |
| **XPHMG_MTX**   | Small matrix and tile compute primitives                          |
| **XPHMG_RT**    | Ray and geometry intersection primitives                          |
| **XPHMG_GFX**   | Texture, interpolation, shading, and visibility helpers           |
| **XPHMG_ACCEL** | Descriptor-driven integration of NPUs and custom accelerators     |

All extensions **consume the same CAP and XMEM state**, ensuring consistent behavior across domains.

---

## ✦ Tile-Based Rendering & Tiled Compute

XPHMG naturally supports **tile-based rendering and tiled compute** through:

* Explicit LDS/XMEM allocation,
* Tiered memory residency (Tier-0 / Tier-1),
* Domain-controlled ownership and handoff,
* Canonical tile layouts shared across engines.

Tiles are **software-defined working sets**, not architectural objects.
Their size, lifetime, and layout are fully controlled by software.

---

## ✦ Versioning & Stability

* **Current version:** **v0.1.0**
* **Status:** Stable architectural baseline

Version 0.1 defines:

* the CAP authority model,
* unified memory semantics,
* execution-domain composition rules,
* instruction-level contracts for all extensions.

Future revisions may extend capabilities, but must preserve the guarantees defined by this baseline unless explicitly stated.

---

## ✦ Repository Contents

```
/
├─ XPHMG.md                 # Root architecture overview
├─ XPHMG_CAP.md             # Capabilities & numeric/memory policy
├─ XPHMG_XMEM.md            # Explicit memory model
├─ XPHMG_RSV.md             # Scalar-vector execution model
├─ XPHMG_RSV-Profiles.md    # Optional RSV instruction profiles
├─ XPHMG_MTX.md             # Matrix & tile compute
├─ XPHMG_RT.md              # Ray traversal & intersection
├─ XPHMG_GFX.md             # Graphics & shading helpers
├─ XPHMG_ACCEL.md           # Descriptor-driven accelerators
```

All documents are part of the **XPHMG v0.1 specification**.

---

## ✦ Intended Audience

XPHMG is intended for:

* Hardware architects and RTL designers
* Compiler and toolchain developers
* Graphics and compute researchers
* Experimental SoC and accelerator projects
* Anyone interested in explicit, low-level graphics/compute execution models

It is **not** intended as an end-user programming API.

---

## ✦ License

The XPHMG specification is released under **CC-BY-SA 4.0**.

---

## ✦ Status

This repository represents the **first stable public release** of XPHMG.

Feedback, discussion, and experimentation are welcome — especially around:

* execution-domain composition,
* memory semantics,
* tile-based and hybrid pipelines,
* accelerator integration.
