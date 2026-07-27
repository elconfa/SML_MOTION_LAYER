**English** | [Italiano](IMPORT_CHECKLIST.it.md)

# SML v9 — CoDeSys import checklist (first Build)

**Goal:** bring `SML_v9/*.txt` into a CoDeSys project with the smoothest possible first Build. The
`.txt` files are **textual ST exports** (declaration + implementation), to be **pasted** into
manually-created objects — they are not importable project files.

Convention: for DUT/GVL/interfaces the file is all "declaration". For POUs (FB/FUNCTION/PROGRAM) the
file contains the header + the VARs (declaration) and the ST body (implementation): split them into the
editor's two panes.

---

## Two projects: library + application

You can import everything into a **single project** (the order in §1 below works as-is), or split it into a
reusable **library** + a per-machine **application** (recommended, see [`../library/README.md`](../library/README.md)).

**Library project** (reusable, no instances) — §1a…1h except `GVL_App`:
`GVL_SML_CONST`, all enums (`SML_DiagCode`, `E_PROGRESS`, `E_AXIS_CTRL`, `E_AXIS_STATE`), all structs
(`OpenSML_Axis`, `ST_CiA402_Status`, `ST_AXIS_CTRL`, `ST_MOVE_DATA`, `ST_AXIS_STATE`, `ST_AXIS_INFO`,
`ST_DriveIn`, `ST_DriveOut`), functions (`f_GetProgress`, `f_GetState`), `I_Axis`, the leaf FBs
(`SML_Power/Reset/Home/ProfilePosition/ProfileVelocity/ProfileVelocity_Jog/Stop/Status/Diagnostics/TouchProbe`,
`FB_S7RTT_OTG`), `FB_AxisCtrl` (+ methods), the UNIONs (`U_AXIS_CTRL/MOVE_DATA/AXIS_STATE/AXIS_INFO`), and the bridge FBs
(`FB_IoLink_In/_Out`, `FB_Mapping_In/_Out`).

**Application project** (created per machine; NOT in the library):
`GVL_App` (`MAX_AXIS`), `GVL_AXIS` (instance arrays + `Control : FB_AxisCtrl` + `ItfSmlAxis`), and `MAIN`
(which instantiates the library bridge FBs). Optionally the image GVLs `GVL_IO` (I/O bridge) and
`GVL_AXIS_MAP` (bus MAPPING). Examples (`PLC_APP`, `FB_AxisCycleDemo`) and test benches (`PRG_*_Test`) live
here too.

> With the library packaged under namespace `SML`, prefix its references in the application
> (`SML.FB_AxisCtrl`, `SML.AXIS_MOVE_ABS`, …). The §1 order below is for the single-project route.

---

## 0. Pre-flight

- [ ] **CoDeSys version** 3.5 (or TwinCAT 3, being CoDeSys-based).
- [ ] **FB_S7RTT_OTG** available: it's included as `.txt`, but it depends on the trajectory generator
      (Ruckig/Struckig light). Verify it compiles standalone.
- [ ] **Standard library** referenced (TON/TOF, CONCAT, SIZEOF, MEMSET…).
- [ ] Decide the first-Build mode: **simulation** (no HW, use the test benches) — recommended for the
      first Build.
- [ ] MAPPING **off** for the first Build: `GVL_AXIS_MAP.AXIS_MAP_ENABLE = FALSE` (runtime flag; MAPPING
      auto-bypasses → behavior as in v7). NB: no `{IF defined(...)}` pragma in the declaration (not
      portable on CoDeSys).
- [ ] Note: `SML_TC3Link` is **TwinCAT-specific** (bridge to the TwinCAT I/O tree). Not needed in
      simulation; on plain CoDeSys the I/O maps differently.
- [ ] The bridge FBs (`FB_IoLink_*`, `FB_Mapping_*`) use `ARRAY[*]` variable-length `VAR_IN_OUT` —
      supported on CoDeSys 3.5 and TwinCAT 3. Only relevant if you use the I/O bridge / MAPPING.

---

## 1. Import order (dependencies resolve bottom-up)

Create the objects in THIS ORDER (Add Object → indicated type):

### 1a. Constants — **first** (used as array bounds)
- [ ] `GVL_SML_CONST`  → GVL  (library: `PROGRESS_SPAN`, `MAP_SIZE_*`)
- [ ] `GVL_App`        → GVL  (application: `MAX_AXIS` — array bound)

### 1b. Enums (DUT → Enumeration)
- [ ] `SML_DiagCode` (if not already present from the base)
- [ ] `E_PROGRESS`
- [ ] `E_AXIS_CTRL`
- [ ] `E_AXIS_STATE`

### 1c. Structs (DUT → Structure)
- [ ] `OpenSML_Axis`, `OpenSML_Control` (base, if not present)
- [ ] `ST_CiA402_Status`  (used by ST_AXIS_INFO)
- [ ] `ST_AXIS_CTRL`
- [ ] `ST_MOVE_DATA`
- [ ] `ST_AXIS_STATE`  (uses `SML_DiagCode`, `E_PROGRESS`, `E_AXIS_CTRL`)
- [ ] `ST_AXIS_INFO`   (uses `ST_CiA402_Status`)

### 1d. Functions (POU → Function)
- [ ] `f_GetProgress`  (uses `E_PROGRESS`, `GVL_SML_CONST`)
- [ ] `f_GetState`

### 1e. Interface (POU → Interface) — see §2
- [ ] `I_Axis` + its child METHODs/PROPERTYs

### 1f. CiA402 leaf FBs (execution level) — the library base
**Minimum** set required by `FB_AxisCtrl` (v9 includes Status + touch-probe):
- [ ] `SML_Power`, `SML_Reset`, `SML_Home`, `SML_ProfilePosition`,
      `SML_ProfileVelocity`, `SML_ProfileVelocity_Jog`, `SML_Stop`,
      `SML_Diagnostics`, `SML_Status`, `SML_TouchProbe`, `FB_S7RTT_OTG`
**Legacy** objects (in `legacy/`, not instantiated — import only if you specifically need them, e.g. the
TwinCAT bridge `SML_TC3Link`):
- [ ] `SML_SyncPosition`, `SML_SyncVelocity`, `FB_SML`, `SML_AxisController`,
      `SML_TC3Link`

### 1g. Control FB (POU → Function Block) — see §2
- [ ] `FB_AxisCtrl` (`IMPLEMENTS I_Axis`) + its METHODs/PROPERTYs

### 1h. UNIONs (DUT → Union)
- [ ] `U_AXIS_CTRL`, `U_MOVE_DATA`, `U_AXIS_STATE`, `U_AXIS_INFO`
      (use `MAP_SIZE_*` from `GVL_SML_CONST`)

### 1i. Hardware I/O bridge (optional — only if you use the bridge, see GUIDA_IO_Linking)
- [ ] `ST_DriveOut`, `ST_DriveIn`  (DUT → Structure) — *library*
- [ ] `FB_IoLink_In`, `FB_IoLink_Out`  (POU → Function Block; *library*, instantiated in MAIN)
- [ ] `GVL_IO`  (GVL; *application*: `DriveOut`/`DriveIn` plain arrays + `IO_LINK_ENABLE`; no `AT` → avoids C0128, map in the configurator)

### 1j. GVL and orchestration
- [ ] `FB_Mapping_In`, `FB_Mapping_Out`  (POU → Function Block; *library*, instantiated in MAIN)
- [ ] `GVL_App`       (GVL; *application*: `MAX_AXIS`)
- [ ] `GVL_AXIS`      (*application*; uses FB_AxisCtrl, I_Axis, ST_*, OpenSML_Axis, MAX_AXIS)
- [ ] `GVL_AXIS_MAP`  (*application*; uses the UNIONs)
- [ ] `MAIN`         (POU → Program; *application*: instantiates the bridge FBs + runs the axis loop)

### 1k. Test benches (POU → Program) — development
- [ ] `PRG_LevelA_Test`, `PRG_LevelB_Test`, `PRG_MultiAxis_Test`

---

## 2. The I_Axis methods (critical point)

`FB_AxisCtrl` declares `IMPLEMENTS I_Axis`: it **won't compile** until all the interface's
methods/properties exist as child objects of the FB.

1. [ ] Create the INTERFACE `I_Axis`. For every entry in `I_Axis.txt` add a child **Method**
       (Enable/Disable/Reset/Home/MoveAbsolute/MoveRelative/MoveVelocity/Jog/MoveFollow/Stop) and a
       child **Property** with a **Get** (Position, Enabled). Paste the signatures (return type +
       VAR_INPUT).
2. [ ] On `FB_AxisCtrl`: right-click → **Implement interfaces…** (generates the method/property stubs)
       **or** add them by hand as child Method/Property.
3. [ ] Paste the **real bodies** from `FB_AxisCtrl_METHODS.txt` into each corresponding method/Get.

---

## 3. Project settings

- [ ] **Non-strict enums**: the enums have no `{attribute 'strict'}`. The code uses `TO_INT(enum)`
      (combined state) and assigns INT→enum in `f_GetProgress`: with non-strict enums it compiles. If
      the project forces strict enums (attribute or option), adapt `f_GetProgress` (explicit mapping)
      and the `TO_INT`s.
- [ ] **MAPPING flag** `GVL_AXIS_MAP.AXIS_MAP_ENABLE`: **FALSE** for the first Build. To enable mapping
      later: set it TRUE (design-time or online). No conditional-compile pragma (`{IF defined(...)}` is
      not supported in the declaration part on CoDeSys). For compile-time exclusion, optional: wrap the
      BODY of FB_Mapping_In/Out in `{IF defined (AXIS_MAP)} ... {END_IF}` in the IMPLEMENTATION.
- [ ] Libraries: no dependency on `memcpy` (copy via assignment). Only Standard + the optional OTG lib
      are needed.

---

## 4. Task configuration

- [ ] Add **ONLY ONE** caller program to the cyclic task:
      - production/HW: `MAIN` (+ I/O bridge for `GVL_AXIS.Axis[]`);
      - simulation: **one** of the benches (`PRG_MultiAxis_Test` or `PRG_LevelB_Test`).
      **DON'T** put MAIN and a bench together (double-call of the same FBs).
- [ ] Set the task cycle time and align `Ctrl.CycleTime` (or `GVL_AXIS.Ctrl[n].CycleTime`) to the
      **same** value (OTG correctness).

---

## 5. Typical first-Build errors and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `FB_AxisCtrl` doesn't implement I_Axis | missing methods/properties | create the Method/Property (§2) |
| `FB_S7RTT_OTG` not found | OTG not imported/licensed | import the OTG FB/library |
| INT→E_PROGRESS conversion not allowed | enum in strict mode | see §3 (adapt) |
| `GVL_App.MAX_AXIS` invalid as bound | constant not resolved | import `GVL_App` (and `GVL_SML_CONST`) first |
| `{IF defined}` pragma not supported in declaration | conditional-compile in VAR | use the `AXIS_MAP_ENABLE` flag (already applied) or move the pragma into the body |
| `CONCAT`/`TON` undefined | Standard not referenced | add the Standard library |

---

## 6. Verification (after a clean Build)

1. [ ] **Build**: zero new errors; `E_*`, `ST_*`, `U_*`, `I_Axis`, `GVL_*`, `FB_AxisCtrl`, `MAIN`,
       `FB_Mapping_*` resolved.
2. [ ] **Level A** — task with `PRG_LevelA_Test`: `xTestPassed → TRUE`
       (`eProgress` INVALID→BUSY→DONE→ERROR→INVALID).
3. [ ] **Level B single-axis** — `PRG_LevelB_Test`: `xTestPassed → TRUE`; observe `State.eState`
       (e.g. 306 = IDLE+DONE, 502 = MOVING+BUSY) and `eStProg = eProgress`.
4. [ ] **Multi-axis** — `PRG_MultiAxis_Test`: `xTestPassed → TRUE`, `xIndependent` TRUE (axis 1 CSP and
       axis 2 JOG in parallel).
5. [ ] **MAPPING** — set `GVL_AXIS_MAP.AXIS_MAP_ENABLE := TRUE`, link/force
       `GVL_AXIS_MAP.Ctrl[n].stData.eCmd` and verify propagation to `GVL_AXIS.Ctrl[n]` and the return
       into `GVL_AXIS_MAP.State[n].stData`.
6. [ ] **Size check**: temporarily lower a `MAP_SIZE_*` below the SIZEOF of the struct → the mapping
       must block (no copy).
