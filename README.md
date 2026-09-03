**English** | [Italiano](README.it.md)

# SML_MOTION_LAYER

**A license-free motion layer for CiA402 drives — CoDeSys 3.5 & TwinCAT 3, pure IEC 61131-3 Structured Text.**

Control EtherCAT servo/stepper drives that speak the **CiA402** profile (homing, profile position/velocity,
jog, cyclic-synchronous position with a jerk-limited online trajectory generator) **entirely in Structured Text** —
one function block per axis, commanded through a data struct *or* an interface. No vendor motion runtime underneath.

> **Built on two existing projects:** the CiA402 execution FBs of
> **[OpenSML](https://github.com/feecat/opensml)** (feecat) — of which SML is a derivative, hence GPL-3.0 — and the
> architectural pattern of **[PLC_MOTION_LAYER](https://github.com/haud-ba/PLC_MOTION_LAYER)** (haud-ba).
> See [Origin & credits](#origin--credits).

![License: GPL-3.0](https://img.shields.io/badge/license-GPL--3.0-blue.svg)
![Platform](https://img.shields.io/badge/PLC-CoDeSys%203.5%20%7C%20TwinCAT%203-informational)
![Language](https://img.shields.io/badge/IEC%2061131--3-Structured%20Text-orange)
![Status](https://img.shields.io/badge/status-early%20%2F%20simulation--tested-yellow)

> **Sources are Structured Text text exports (`.txt`)** to be imported into a CoDeSys/TwinCAT project.
> See [`docs/IMPORT_CHECKLIST.md`](docs/IMPORT_CHECKLIST.md).

---

## 💡 No axis/motion license required

This is the headline. SML implements the **CiA402 drive state machine** and the **motion profiles** itself, in
plain IEC 61131-3 ST, commanding the drive **directly over the EtherCAT process image** (PDOs). Therefore:

- **CoDeSys** — you do **not** need **SoftMotion** (the paid, often per-axis motion add-on).
- **TwinCAT** — you do **not** need **TF5000 TC3 NC PTP** (the PLCopen `MC_*` motion library).

You keep only what you already have: the standard **PLC runtime** and an **EtherCAT master**. Motion — the part that
normally costs a license — lives in this open-source code.

> Honest scope: a base PLC-runtime license and the fieldbus still apply as usual on your platform. The point is that
> **no *motion* option** is needed. And, as with any motion code, **you are responsible for validating it on your own
> hardware and for machine safety** — see [Maturity & safety](#maturity--safety).

---

## Why you might want it

- **License-free motion** on both CoDeSys and TwinCAT (see above).
- **One entry point per axis** — `FB_AxisCtrl`: a command enum (`eCmd`) plus a clean `Ctrl / Data / State / Info`
  data contract. It's a **superset** of the classic profile FBs: PP/PV/Jog **and** CSP+OTG, **status decoding**
  and **touch-probe** all in one block.
- **Two ways to command it**, use whichever fits your style: a **data struct** (great for HMI/recipe-driven code) or
  an **interface** `I_Axis` (great for high-level coordinators).
- **Portable** — the *same* ST runs on CoDeSys and TwinCAT; only the I/O mapping to the drive differs.
- **Unified status model** — an `E_PROGRESS` progress enum and a combined `E_AXIS_STATE` so reading "where is this
  axis" is one field, not a scavenger hunt across flags.
- **Multi-axis by design** — an `ARRAY[1..MAX_AXIS]` orchestrated by one `MAIN`; scale by changing a single constant.
- **Bus-agnostic MAPPING layer** (UNION-based) to expose axis Ctrl/State over a fieldbus or ADS with a fixed-size raw
  image — optional, off by default.
- **Simulate without hardware** — test benches drive a CiA402 emulator, so you can bring the logic up before wiring.
- **Library + application split** — a reusable `library/` (namespace `SML`) and an `application/` template with a
  real, worked **2-axis example machine**.

---

## Architecture

```
   YOUR APPLICATION   (writes Ctrl + Data, reads State + Info)
        │
        ▼
   MAIN  ──►  FOR n := 1..MAX_AXIS ──►  GVL_AXIS.Control[n]  (FB_AxisCtrl)
                                             │  drives the CiA402 leaf FBs
                                             ▼
        Power / Reset / Home / ProfilePosition / ProfileVelocity /
        Jog / Stop / Status / Diagnostics / TouchProbe / CSP+OTG
                                             │
                                             ▼
   GVL_AXIS.Axis[n]  (OpenSML_Axis = CiA402 PDO image)  ──►  EtherCAT DRIVE
```

---

## Usage in 30 seconds

Command axis 1 through the **data struct**:

```pascal
// Enable, then home (issue commands as the axis becomes ready)
GVL_AXIS.Ctrl[1].eCmd := AXIS_ENABLE;
GVL_AXIS.Ctrl[1].eCmd := AXIS_HOME;

// Absolute move: go to 250.0 units at 100.0 units/s
GVL_AXIS.Data[1].lrTargetPosition := 250.0;
GVL_AXIS.Data[1].lrVelocity       := 100.0;
GVL_AXIS.Ctrl[1].eCmd             := AXIS_MOVE_ABS;

// Read where it is
IF GVL_AXIS.State[1].eProgress = PROGRESS_DONE THEN
    // move finished — GVL_AXIS.Info[1].lrActualPosition is at target
END_IF
IF GVL_AXIS.State[1].xError THEN
    GVL_AXIS.Ctrl[1].eCmd := AXIS_RESET;
END_IF
```

Or the exact same move through the **interface** `I_Axis`:

```pascal
GVL_AXIS.ItfSmlAxis[1].Enable();
GVL_AXIS.ItfSmlAxis[1].Home();
GVL_AXIS.ItfSmlAxis[1].MoveAbsolute(lrPosition := 250.0, lrVelocity := 100.0);
```

Continuous velocity, and a jerk-limited following move (CSP + OTG):

```pascal
GVL_AXIS.Data[1].lrVelocity := 80.0;                  // signed
GVL_AXIS.Ctrl[1].eCmd       := AXIS_MOVE_VELOCITY;

GVL_AXIS.ItfSmlAxis[1].MoveFollow(...);               // CSP with online trajectory generator
```

Commands (`eCmd`): `AXIS_ENABLE`, `AXIS_DISABLE`, `AXIS_RESET`, `AXIS_HOME`, `AXIS_MOVE_ABS`, `AXIS_MOVE_REL`,
`AXIS_MOVE_VELOCITY`, `AXIS_JOG_POS`, `AXIS_JOG_NEG`, `AXIS_MOVE_CSP`, `AXIS_STOP`.

Full variable list: [`docs/API_Reference.md`](docs/API_Reference.md). Command/usage guide:
[`docs/MANUALE_SML.md`](docs/MANUALE_SML.md).

---

## Repository layout (two levels)

| Folder | What's in it |
|---|---|
| **`library/`** | reusable core (enums, DUTs, `I_Axis`, `FB_AxisCtrl`, CiA402 leaf FBs, functions, OTG) — [README](library/README.md) |
| **`application/`** | machine template that references the library — [README](application/README.md) |
| `application/src/` | `GVL_App` (`MAX_AXIS`), `GVL_AXIS`, `MAIN`, MAPPING, optional I/O bridge |
| `application/examples/` | `FB_AxisCycleDemo`, `PLC_APP` (the 2-axis machine) |
| `application/tests/` | simulation benches (CiA402 emulator) |
| `legacy/` | superseded facades, preserved for provenance |
| `docs/` | [API reference](docs/API_Reference.md), manual, I/O linking guide, import checklist, design note |
| `binaries/` | compiled CoDeSys (`codesys/`) and TwinCAT (`twincat/`) projects (Git LFS) |

---

## Getting started

1. Build the **library** from `library/src/` (namespace `SML`) — see [`library/README.md`](library/README.md) — or
   import everything into a single project following [`docs/IMPORT_CHECKLIST.md`](docs/IMPORT_CHECKLIST.md).
   Create the `I_Axis` methods/properties under `FB_AxisCtrl` from `FB_AxisCtrl_METHODS.txt`.
2. In the **application**, reference the library and set `GVL_App.MAX_AXIS`.
3. Put `MAIN` in a cyclic task.
4. Map `GVL_AXIS.Axis[n]` to the drive PDOs — see [`docs/GUIDA_IO_Linking.md`](docs/GUIDA_IO_Linking.md).
5. Command with `GVL_AXIS.Ctrl[n].eCmd := ...` and read `GVL_AXIS.State[n]` / `Info[n]`.

**Try it without a drive:** set `Ctrl[n].xSimulation := TRUE`, or run one of the benches in `application/tests/`.

---

## Worked example — `application/examples/PLC_APP`

A 2-axis measure-and-sort machine:

- **Axis 1** = a continuously rotating belt (velocity mode) with **touch-probe**;
- **Axis 2** = a positioner.

Cycle: home once → measure each piece via touch-probe between "photocell covered" and "photocell cleared" → if the
piece is **out of tolerance**, a **non-blocking reject** (belt keeps running, the piece is ejected and counted by a
parallel tracker, and the **next piece is already entering** — pipelined) → if **good**, axis 2 goes to +400 then
back to 0 while the belt restarts → **standby** with timeout (re-home with a command, resume on an external signal).
Clean machine I/O in and out. Details in [`docs/MANUALE_SML.md`](docs/MANUALE_SML.md) (Appendix B).

---

## Maturity & safety

Be aware of what this is:

- **Young project, simulation-tested.** The core compiled cleanly in CoDeSys 3.5 and runs against a CiA402 emulator
  in the test benches. The recent two-level reorganization and object renaming are **import-ready** but should be
  re-verified with a Build after you import them (the sources are `.txt` exports, so there's no binary to trust yet).
- **Motion control is safety-relevant.** This code moves real hardware. **Validate every function on your own drive**,
  keep a hardware **STO / emergency-stop** path independent of this logic, and respect the machinery safety rules that
  apply to you. The license disclaims warranty (GPL-3.0, "as is").

Feedback, issues and real-world reports are very welcome — this is exactly the stage where they help most.

---

## Origin & credits

This project stands openly on two shoulders, and it was built with the help of an AI assistant. None of that is hidden:

- **[OpenSML](https://github.com/feecat/opensml)** (feecat) — the "SoftMotion Light" CiA402 execution FBs this layer
  drives. SML is a **derivative of OpenSML**, which is why this repo is **GPL-3.0** (see [License](#license)).
- **[PLC_MOTION_LAYER](https://github.com/haud-ba/PLC_MOTION_LAYER)** (haud-ba) — the **architectural pattern**
  (Ctrl/State/Info/Data contract, combined progress state, per-axis FB, array orchestration, UNION mapping) was
  extracted from this TwinCAT NC/PLCopen project and re-applied on top of SML.
- **AI-assisted development.** The design extraction, the porting to SML, the multi-axis and MAPPING layers, the
  example machine and this documentation were produced in collaboration with **Anthropic's Claude**. A human reviewed
  and directed the work; the AI wrote and refactored code and docs under that direction. The value on offer is the
  *integration* — a coherent, license-free motion layer for two runtimes — not a claim of hand-crafted novelty.

So: what's new here is putting OpenSML's execution FBs behind PLC_MOTION_LAYER's contract, on both CoDeSys and TwinCAT,
with no motion license. The OTG (`FB_S7RTT_OTG`) and any vendor libraries keep their own licenses.

---

## License

**GPL-3.0** — see [`LICENSE`](LICENSE). Because SML derives from OpenSML (GPL-3.0), the whole work is GPL-3.0 by
copyleft (a permissive license would not be compatible, and a closed compiled-library redistributed to third parties
would violate the GPL).

> Copyright (C) 2024-2026 Massimo Confalonieri. This is free software; you can redistribute it and/or modify it under
> the terms of the GNU General Public License version 3 as published by the Free Software Foundation.
