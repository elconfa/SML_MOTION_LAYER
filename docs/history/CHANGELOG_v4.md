# SML v4 — Audit & Changelog

**Base:** SML_v3 (merged OpenSML + Confa)
**Date:** 2026-06-19
**Scope:** deep audit of the v3 source code (not just the analysis docs) +
new runtime diagnostics subsystem.

The v3 `_Analysis.md` fixes were verified to be **actually present** in the
v3 code (E-stop direct polarity, `Target_Position := Position_Actual_Value`
on error, `Scale=0` guard, `ABS()` on xMoving, etc.). The audit then found
**new defects in the code** that were not documented. All are fixed in v4.

---

## Fixes applied in v4

### 🔴 Critical
- **C1 — Sync FBs did not compile.** `SML_SyncPosition`/`SML_SyncVelocity`
  called the OTG with `CycleTime := ...` but the formal parameter is
  `CycleTIme` (intentional typo). CoDeSys error *"CycleTime non è un ingresso
  di FB_S7RTT_OTG"*. Fixed to `CycleTIme :=` in both callers.

### 🟠 High
- **A1 — Granular fault reset.** In v3, `FB_SML.RstEx` only reset the drive
  (`SML_Reset`); the motion sub-FBs' `xReset` inputs were unconnected, so a
  following-error stayed latched until power dropped. Now `RstEx` also pulses
  `xReset` on Home / MoveHome / MoveAbs / MoveVel / Jog / TouchProbe.
- **A2 — Command arbiter.** Multiple motion FBs share one `Axis` struct.
  v3 had no interlock, so simultaneous commands corrupted `ControlWord` /
  `Target_*`. v4 adds a fixed-priority arbiter (Jog > Vel > MoveAbs > MoveHome)
  that masks lower-priority executes. Halt stays independent (only bit 8).

### 🟡 Medium
- **M1 — Deterministic OTG seed.** `FB_S7RTT_OTG` re-initialized whenever
  `CurrentPosition` changed *by value* — fragile (could re-init every cycle and
  never advance). Replaced with an explicit `xInit` input. Callers assert
  `xInit=TRUE` and run the OTG **in the mode-switch state** so re-entry after an
  error always reseeds from the real position.
- **M2 — Simulation seed.** `xSimulation` paths jumped state 0 → 20, skipping
  the seed state. Now they pass through state 10 (with `OR xSimulation` on the
  transition) so the OTG is seeded in simulation too.
- **M3 — Quick-stop recovery.** `SML_Power` state 50 only watched the Fault
  bit. After E-stop release the drive could stay in `QuickStopActive`
  (StatusWord bit 5 = 0) with a misleading `xDone`. Now it detects this and
  re-runs the enable sequence (drops `xDone` until re-confirmed).
- **M4 — No needless disable/enable.** `SML_ProfileVelocity` /
  `SML_ProfileVelocity_Jog` forced a `ControlWord.3 := FALSE` cycle on every
  start even when already enabled in the correct mode. Added a fast path that
  enters state 20 directly when already `OperationEnabled` in PV mode.
- **M5 — Diagnostics exposed.** `FB_SML` now propagates `FollowingError`,
  `Status_InternalLimit`, `Status_VelocityMode`, `Status_SyncVelocityMode`
  (computed by `SML_Status` but dropped in v3).

### 🟢 Low
- **B1** — `FB_SML.inst_TouchProbe.xReset` now connected (was unconnected).
- **B2** — `FB_SML.HomePos` unit comment corrected to "encoder counts → 0x607C".
- **B3** — `SML_AxisController` home offset no longer hardcoded to 0; new
  `OpenSML_Control.HomeOffset` field.
- **B4** — Removed the redundant signed `LREAL_TO_DINT` branch in the CSP
  conversions (CoDeSys handles negatives correctly).

---

## New: runtime diagnostics subsystem

### `SML_DiagCode` (ENUM, UINT)
Categorized codes: `0` OK, `1..9` warnings, `10..19` drive faults,
`20..29` library/motion-FB faults (incl. `DIAG_OTG_ERROR := 26`).

### `SML_Diagnostics` (FB)
Aggregates drive StatusWord bits + `Error_Code` (0x603F) + per-FB error flags
into a single clear picture:
- `DiagCode` / `DiagText` — live highest-priority condition.
- `FirstFaultCode` / `FirstFaultText` — **latched first cause** (survives
  follow-on faults; cleared by `xReset`).
- `xActiveFault` / `xActiveWarning` — clean fault-vs-warning classification.
- `DriveErrorCode` / `FollowingError` — surfaced raw drive detail.
- `FaultCount` — cumulative fault-event counter for maintenance.

Priority: Drive fault → Quick-stop → Following error → Power → Home → Move →
Velocity → Jog → Touch-probe → Internal limit (warn) → Warning → OK.

### Integration
- **`FB_SML`** (PP/PV/Jog path): `SML_Diagnostics` built in as `inst_Diag`.
  Outputs exposed as FB_SML outputs; pulse `DiagReset` to acknowledge.
- **`SML_AxisController`** (CSP/OTG path): `SML_Diagnostics` built in as `Diag`,
  called after the state machine so `otg.xError` is current. Results are written
  to the `OpenSML_Control` struct (`DiagCode`, `DiagText`, `xActiveFault`,
  `xActiveWarning`, `FirstFaultCode/Text`, `DriveErrorCode`, `FollowingError`,
  `FaultCount`); pulse `Control.xDiagReset` to acknowledge. The OTG constraint
  error feeds `xOtgError` → `DIAG_OTG_ERROR`.

Both facades now give the operator one clear diagnostic picture without any
extra wiring in the application.

---

## File inventory (19 files)

New in v4: `SML_DiagCode.txt`, `SML_Diagnostics.txt`.
Modified: `FB_S7RTT_OTG`, `FB_SML`, `OpenSML_Control`, `SML_Power`,
`SML_ProfileVelocity`, `SML_ProfileVelocity_Jog`, `SML_SyncPosition`,
`SML_SyncVelocity`, `SML_AxisController`.
Unchanged from v3: `OpenSML_Axis`, `SML_Home`, `SML_ProfilePosition`,
`SML_Reset`, `SML_Status`, `SML_Stop`, `SML_TouchProbe`, `SML_TC3Link`.

---

## Still to verify on hardware (carried from v3)
1. `StatusWord` bit 14 (standstill) and bit 10 (Target Reached) meaning —
   drive-dependent.
2. `SML_Status.Status_Homed` reads `Digital_Inputs.0` — needs DI0 configured.
3. `STANDSTILL_THRESHOLD` (SML_Status) in drive velocity units.
4. Quick-stop option code `0x605A` behaviour for the M3 recovery path.
5. `CycleTime` must equal the real PLC task period (OTG accuracy).
6. Enum relational comparison (`DiagCode >= DIAG_DRIVE_FAULT`) is valid for
   UINT-backed enums in CoDeSys; verify on your target compiler.
