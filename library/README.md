**English** | [Italiano](README.it.md)

# SML — Library (reusable core)

The **library** component of the Motion Layer: reusable objects only — **no instances, no machine
configuration, no PROGRAMs**. Package it as a *CoDeSys Library* / *TwinCAT Library*, then reference it
from each PLC application.

The library is **self-contained**: every object here depends only on other objects in this folder plus
the **Standard** IEC library (`TON`, and the operators `TON/ABS/MIN/MAX/SQRT/SIZEOF/…`). Verified: no
object references `MAX_AXIS`, the instance GVLs, or any legacy/application file in code.

---

## Manifest (set these in *Project Information* when you create the library)

| Field | Value |
|---|---|
| Title | `SML` |
| Subtitle | SoftMotion Light — CiA402 Motion Layer |
| Namespace | `SML` |
| Version | `0.9.0` (pre-1.0: early, simulation-tested) |
| Company / Author | *(set your own)* |
| License | GPL-3.0 (see [`../LICENSE`](../LICENSE)) |
| Placeholder | leave default (or `SML, * (SML)`) |

## Dependencies

- **Standard** (CoDeSys) / **Tc2_Standard** (TwinCAT) — for the IEC standard FB `TON` (and `TOF`/`F_TRIG`
  only in the *application* files, not here). Normally already referenced in a new standard project.
- Everything else is **built-in** to the compiler: type conversions (`LREAL_TO_DINT`, …), math operators
  (`ABS/MIN/MAX/SQRT`), `SIZEOF`, `STRING`. No pointers, no `MEMCPY`/`MEMSET`, no string-function library.
- **No motion library**: no SoftMotion / `SM3_` / `SMC_`, no `Tc2_MC2` / NC, no `MC_*`, no `AXIS_REF`.
  This is the technical reason there is **no axis/motion license** requirement.
- `FB_S7RTT_OTG` (jerk-limited trajectory generator) is **included here as source** — not an external
  dependency; it uses only built-in math.

---

## Exported objects (`src/`) — 37 objects, in build order

Create the objects **bottom-up** (each line depends only on the ones above):

1. **Constants** — GVL
   - `GVL_SML_CONST` (`PROGRESS_SPAN`, `MAP_SIZE_*`)
2. **Enums** — DUT → Enumeration
   - `SML_DiagCode`, `E_PROGRESS`, `E_AXIS_CTRL`, `E_AXIS_STATE`
3. **Structs** — DUT → Structure
   - `OpenSML_Axis` (CiA402 PDO image), `ST_CiA402_Status`,
     `ST_AXIS_CTRL`, `ST_MOVE_DATA`, `ST_AXIS_STATE`, `ST_AXIS_INFO`,
     `ST_DriveIn`, `ST_DriveOut` (I/O bridge halves)
4. **Functions** — POU → Function
   - `f_GetProgress`, `f_GetState`
5. **Interface** — POU → Interface (+ its child METHODs/PROPERTYs, see below)
   - `I_Axis`
6. **CiA402 leaf FBs** — POU → Function Block
   - `SML_Power`, `SML_Reset`, `SML_Home`, `SML_ProfilePosition`, `SML_ProfileVelocity`,
     `SML_ProfileVelocity_Jog`, `SML_Stop`, `SML_Status`, `SML_Diagnostics`, `SML_TouchProbe`,
     `FB_S7RTT_OTG`
7. **Control FB** — POU → Function Block (`IMPLEMENTS I_Axis`)
   - `FB_AxisCtrl` (+ method/property bodies from `FB_AxisCtrl_METHODS.txt`)
8. **UNIONs** (for the optional MAPPING layer) — DUT → Union
   - `U_AXIS_CTRL`, `U_MOVE_DATA`, `U_AXIS_STATE`, `U_AXIS_INFO`
9. **I/O & MAPPING bridge FBs** (optional, reusable copy loops) — POU → Function Block
   - `FB_IoLink_In`, `FB_IoLink_Out` — drive image ↔ `Axis[]`
   - `FB_Mapping_In`, `FB_Mapping_Out` — bus UNIONs ↔ internal structs (with runtime size check)
   - The arrays are passed by `VAR_IN_OUT` as `ARRAY[*]` (variable length): the application instantiates
     these FBs in `MAIN` and passes its own GVL arrays + enable flag. (Requires `ARRAY[*]` VAR_IN_OUT
     support — CoDeSys 3.5 and TwinCAT 3.)

> The `.txt` files are **textual ST exports**: create each object by hand (Add Object → the type shown)
> and paste the declaration/implementation. For POUs, split the header+VARs (declaration) and the ST body
> (implementation) into the editor's two panes.

---

## The `I_Axis` methods (the one fiddly step)

`FB_AxisCtrl` declares `IMPLEMENTS I_Axis`, so it won't compile until the interface's methods/properties
exist as **child objects**:

1. Create the INTERFACE `I_Axis`; for each entry in `I_Axis.txt` add a child **Method**
   (Enable/Disable/Reset/Home/MoveAbsolute/MoveRelative/MoveVelocity/Jog/MoveFollow/Stop) and a child
   **Property** with a **Get** (Position, Enabled).
2. On `FB_AxisCtrl`: right-click → **Implement interfaces…** to generate the stubs (or add them by hand).
3. Paste the **real bodies** from `FB_AxisCtrl_METHODS.txt` into each method/Get.

---

## Build the library in CoDeSys

1. *File → New Project → Empty project* (or *Library*). Set *Project → Project Information*:
   Title/Namespace/Version/Company as in the manifest.
2. Add the objects from `src/` in the build order above (create the `I_Axis` children under `FB_AxisCtrl`).
3. Ensure the **Standard** library is referenced (Library Manager) — usually already there.
4. *Build* → zero errors.
5. *File → Save Project as Library* (or *Save as Compiled Library* for a distributable `.compiled-library`),
   then *Install* into the Library Repository.

## Build the library in TwinCAT

1. *PLC → Library Project* (or a standard PLC project you'll save as library). Set the library metadata
   (Title/Version/Company) in the project properties.
2. Add the objects from `src/` in the same build order; reference **Tc2_Standard**.
3. *Build* → *Save as library* → it appears in the *Library Repository*.

---

## Reference it from an application

1. In the application project: *Library Manager → Add library → SML*.
2. Because the library uses namespace `SML`, prefix library references where the compiler asks:
   `SML.FB_AxisCtrl`, `SML.I_Axis`, `SML.AXIS_MOVE_ABS`, `SML.E_PROGRESS.PROGRESS_DONE`, …
3. Create the **application-side** objects (they are NOT in the library — see
   [`../application/README.md`](../application/README.md) and
   [`../docs/IMPORT_CHECKLIST.md`](../docs/IMPORT_CHECKLIST.md)):
   `GVL_App` (`MAX_AXIS`), `GVL_AXIS` (the `[1..MAX_AXIS]` arrays + `Control : SML.FB_AxisCtrl` +
   `ItfSmlAxis : SML.I_Axis`), and `MAIN`. Optionally the image GVLs `GVL_IO` (I/O bridge) and
   `GVL_AXIS_MAP` (bus MAPPING): the copy FBs `SML.FB_IoLink_In/_Out` and `SML.FB_Mapping_In/_Out` are in
   the library — you just **instantiate them in `MAIN`** and pass the GVL arrays + the enable flag.

---

## Versioning

`0.9.0` reflects an early, simulation-tested state (see the root README "Maturity & safety"). Bump the
minor for additive changes, the patch for fixes; reserve `1.0.0` for the first hardware-validated release.
Keep the namespace `SML` stable so applications don't have to re-prefix.
