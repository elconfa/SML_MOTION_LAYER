**English** | [Italiano](MANUALE_SML.it.md)

# SML Manual — Axis Control (CiA402 Motion Layer)

**Version:** SML_v9 · **Target:** CoDeSys 3.5 / TwinCAT 3 · CiA402 over EtherCAT

A user manual for a programmer who doesn't know the project but needs to **command one or more
axes**. If you come from PLCopen (`MC_Power`, `MC_MoveAbsolute`, …): here you **don't** instantiate
a separate FB with dozens of inputs for every function; you write **one command** into a struct and
read the **status** from another. A single FB per axis does everything.

---

## 1. The concept in 1 minute

Each axis is handled by an instance of **`FB_AxisCtrl`**. You interact with **four structures**
(the "data contract"):

```
   APPLICATION
      │  writes Ctrl (command) + Data (setpoint)
      ▼
   ┌───────────────────────────────┐
   │  FB_AxisCtrl  (one per axis)   │  ← executes, drives the CiA402 FBs
   └───────────────────────────────┘
      │  writes State (result) + Info (actual values)
      ▼
   APPLICATION reads
```

- **`Ctrl`** — what the axis must do (`eCmd`) + configuration.  *(you write)*
- **`Data`** — the motion numbers: position, velocity, acc, dec, jerk (user units).  *(you write)*
- **`State`** — result: progress, done/error, diagnostics.  *(you read)*
- **`Info`** — actual values: position, velocity, status bits, touch-probe.  *(you read)*

Command it in **two equivalent ways**:
1. **Struct**: `GVL_AXIS.Ctrl[1].eCmd := AXIS_MOVE_ABS;`
2. **Methods** (interface `I_Axis`): `GVL_AXIS.ItfSmlAxis[1].MoveAbsolute(50.0, 100.0);`

The **`eCmd` command is level-triggered**: it stays active until you change it. One `eCmd` per axis →
one motion at a time (no conflicts).

---

## 2. Quick start (copy-paste)

Axis 1, in simulation or on already-connected hardware:

```pascal
// one-time / configuration
GVL_AXIS.Ctrl[1].Scale     := 4096.0;   // counts per unit (mm, revs, …)
GVL_AXIS.Ctrl[1].CycleTime := 0.001;    // = task period [s]

// 1) enable
GVL_AXIS.Ctrl[1].eCmd := AXIS_ENABLE;
// wait for: GVL_AXIS.Info[1].xEnabled = TRUE  (State.eProgress = PROGRESS_DONE)

// 2) homing
GVL_AXIS.Ctrl[1].eCmd := AXIS_HOME;
// wait for: GVL_AXIS.State[1].xDone = TRUE

// 3) move to 50 mm at 100 mm/s
GVL_AXIS.Data[1].lrTargetPosition := 50.0;
GVL_AXIS.Data[1].lrVelocity       := 100.0;
GVL_AXIS.Ctrl[1].eCmd := AXIS_MOVE_ABS;
// wait for: GVL_AXIS.State[1].xDone = TRUE
```

Note: `MAIN` already calls `FB_AxisCtrl` for all axes every cycle. You only touch
`GVL_AXIS.Ctrl[]`/`Data[]` and read `State[]`/`Info[]`.

---

## 3. `eCmd` command reference (like the MC_* pages)

Set `Ctrl.eCmd` to the value you want. The "DONE when" column tells you when
`State.xDone` becomes TRUE (`State.eProgress = PROGRESS_DONE`).

| eCmd | What it does | Data used | DONE when | Notes |
|---|---|---|---|---|
| `AXIS_NULL` | no command; holds the state | — | — | the axis stays as it is (idle or in error) |
| `AXIS_ENABLE` | enable the drive (Power on) | — | `Info.xEnabled` | equivalent to MC_Power |
| `AXIS_DISABLE` | disable the drive | — | drive not enabled | CiA402 Shutdown |
| `AXIS_RESET` | fault reset (drive + FB) | — | fault cleared | equivalent to MC_Reset |
| `AXIS_HOME` | homing | `HomeOffset` (in Ctrl) | homing complete | MC_Home |
| `AXIS_MOVE_ABS` | absolute move (PP) | `lrTargetPosition`, `lrVelocity`, `lrAcceleration`, `lrDeceleration` | target reached | MC_MoveAbsolute |
| `AXIS_MOVE_REL` | relative move (PP) | as above (`lrTargetPosition` = distance) | target reached | MC_MoveRelative |
| `AXIS_MOVE_VELOCITY` | continuous velocity (PV) | `lrVelocity` (**signed**), `lrAcceleration`, `lrDeceleration` | velocity reached | MC_MoveVelocity |
| `AXIS_JOG_POS` | jog positive (held) | `lrVelocity` (magnitude) | — (continuous motion) | MC_Jog + |
| `AXIS_JOG_NEG` | jog negative (held) | `lrVelocity` (magnitude) | — (continuous motion) | MC_Jog − |
| `AXIS_MOVE_CSP` | CSP+OTG jerk-limited following | `lrTargetPosition`, `lrVelocity`, `lrAcceleration`, `lrJerk` | within `PositionWindow` | **online** re-targetable |
| `AXIS_STOP` | Halt (controlled deceleration) | — | axis stopped | MC_Halt |

**Golden rules**
- To **enable**, any command other than `AXIS_NULL`/`AXIS_DISABLE` is enough (enable is implicit).
  Recommended: an explicit `AXIS_ENABLE` before moving.
- **`AXIS_DISABLE`** removes the enable. **`AXIS_NULL`** keeps it.
- **Discrete move (PP)**: set `Data` **then** `eCmd`. For a **new** discrete target, re-trigger (go
  through `AXIS_NULL` and back to `AXIS_MOVE_ABS`, or issue another command). For **smooth/online**
  re-targeting use `AXIS_MOVE_CSP` (update `lrTargetPosition` while the axis follows).
- **Jog**: hold `AXIS_JOG_POS`/`NEG`; release by setting `AXIS_NULL` or `AXIS_STOP`.

---

## 4. Commanding via the `I_Axis` interface (methods)

An alternative to the struct, handy for coordinators/libraries. The methods are **setters**: they set
`Ctrl.eCmd`+`Data` and return the progress (`E_PROGRESS`). Execution happens in the FB's normal cycle.

```pascal
VAR ax : I_Axis; END_VAR
ax := GVL_AXIS.ItfSmlAxis[1];

ax.Enable();
ax.Home();
ax.MoveAbsolute(lrPosition := 50.0, lrVelocity := 100.0);
ax.MoveVelocity(lrVelocity := -30.0);      // PV, sign = direction
ax.Jog(xForward := TRUE, xReverse := FALSE, lrVelocity := 20.0);
ax.MoveFollow(lrPosition := 120.0, lrVelocity := 80.0);  // CSP online
ax.Stop();
ax.Disable();
ax.Reset();

// properties (read)
rPos := ax.Position;      // actual position [units]
bOn  := ax.Enabled;       // drive enabled
```

Use **either** the methods **or** the struct on the same axis, not both in the same cycle.

---

## 5. Reading the status

### 5.1 `State` — command result
| Field | Type | Meaning |
|---|---|---|
| `eProgress` | `E_PROGRESS` | progress: INVALID / INIT / BUSY / PREPARE / STARTUP / CHECK / **DONE** / **ERROR** |
| `eState` | INT | **combined state** = functional state + progress (see 5.2) |
| `eCmdActive` | `E_AXIS_CTRL` | command currently executing |
| `xDone` | BOOL | current command complete (= `eProgress = DONE`) |
| `xError` | BOOL | motion error (aggregated) |
| `DiagCode` / `DiagText` | `SML_DiagCode` / STRING | live condition + text |
| `FirstFaultCode` / `FirstFaultText` | | first latched fault (root cause) |

### 5.2 Combined state `eState`
`eState = E_AXIS_STATE + E_PROGRESS`. Examples: `306` = `AXIS_STATE_IDLE(300)` +
`PROGRESS_DONE(6)`; `502` = `AXIS_STATE_MOVING(500)` + `PROGRESS_BUSY(2)`.
To decompose it, use the functions:
```pascal
eBase := f_GetState(State.eState);      // 500  (functional part)
eProg := f_GetProgress(State.eState);   // 2    (E_PROGRESS.PROGRESS_BUSY)
```
Functional states: `NULL 0, INIT 100, DISABLED 200, IDLE 300, HOMING 400,
MOVING 500, VELOCITY 600, STOPPING 700, ERROR 900`.

### 5.3 `Info` — actual values
| Field | Meaning |
|---|---|
| `lrActualPosition` / `lrActualVelocity` | actual position / velocity [units] |
| `xEnabled` | drive in OperationEnabled |
| `xHomed` | homing done at least once |
| `xMoving` | axis is moving |
| `xInPosition` | within `PositionWindow` (CSP path) |
| `Status` | full CiA402 decoder (see 5.4) |
| `xTouchProbeDoneR/F`, `TouchProbeRisingValue/FallingValue`, `xTouchProbeBusy`, `xTouchProbeError` | touch-probe results |

### 5.4 `Info.Status` — CiA402 bits (from SML_Status)
`OperationEnabled, SwitchedOn, ReadyToSwitchOn, Fault, Warning, QuickStopActive,
VoltageEnabled, TargetReached, Halted, InternalLimit, Moving, StandStill, Homed,
ProfilePositionMode, ProfileVelocityMode, SyncPositionMode(CSP), HomingMode, …`
plus `Err`, `ErrId` (0x603F), `ActTorque`, `FollowingError`.

---

## 6. Touch-probe (position latch)

It runs **in parallel** with motion (it is not an `eCmd`). Command it via `Ctrl`, read it via `Info`.
```pascal
GVL_AXIS.Ctrl[1].xTouchProbeEnable := TRUE;   // arm the hardware
GVL_AXIS.Ctrl[1].xTouchProbeRising := TRUE;   // arm the rising edge
// ... on trigger:
// Info.xTouchProbeDoneR = TRUE
// Info.TouchProbeRisingValue = latched position [encoder counts]
```
A touch-probe error goes into the **diagnostics** and into `Info.xTouchProbeError`, but it does
**not** put the axis into a motion ERROR state.

---

## 7. Configuration (`Ctrl`)

| Field | Default | Meaning |
|---|---|---|
| `xEmergencyStop` | TRUE | **TRUE = run**, FALSE = quick-stop (CiA402 bit 2). Keep it TRUE to operate |
| `xSimulation` | FALSE | mirror `Target → Actual` (test without a drive) |
| `Scale` | 4096.0 | counts per user unit. **> 0** |
| `CycleTime` | 0.001 | task period [s]. Must match (for the OTG) |
| `PositionWindow` | 0.01 | "in position" deadband [units] (CSP) |
| `HomeOffset` | 0 | homing offset [encoder counts → 0x607C] |
| `xTouchProbe*` | FALSE | touch-probe control (see 6) |

`Data`: `lrTargetPosition` [units], `lrVelocity` [units/s, signed for PV],
`lrAcceleration`, `lrDeceleration` [units/s²], `lrJerk` [units/s³, CSP only].

---

## 8. What each file does

### Data contract (DUT)
| File | Role |
|---|---|
| `ST_AXIS_CTRL` | command (`eCmd`) + configuration. App → layer |
| `ST_MOVE_DATA` | motion setpoints in user units. App → layer |
| `ST_AXIS_STATE` | result + combined state + diagnostics. Layer → app |
| `ST_AXIS_INFO` | actual values + CiA402 Status + touch-probe. Layer → app |
| `ST_CiA402_Status` | all CiA402 status bits (inside `Info.Status`) |

### Enums (DUT)
| File | Role |
|---|---|
| `E_AXIS_CTRL` | the commands (`eCmd`) |
| `E_AXIS_STATE` | functional states (×100, for the combined state) |
| `E_PROGRESS` | unified progress (INVALID…DONE/ERROR) |
| `SML_DiagCode` | categorized diagnostic codes |

### Control
| File | Role |
|---|---|
| `FB_AxisCtrl` | **the axis control FB**: translates `eCmd` into calls to the leaf FBs + CSP/OTG; fills State/Info; implements `I_Axis` |
| `FB_AxisCtrl_METHODS` | body of the `I_Axis` methods/properties (child objects of the FB) |
| `I_Axis` | method interface (Enable/Home/MoveAbsolute/…) |

### Execution (CiA402 leaf FBs — driven by FB_AxisCtrl)
| File | Role (≈ PLCopen) |
|---|---|
| `SML_Power` | drive enable (≈ MC_Power) |
| `SML_Reset` | fault reset (≈ MC_Reset) |
| `SML_Home` | homing (≈ MC_Home) |
| `SML_ProfilePosition` | absolute/relative move, PP mode (≈ MC_MoveAbsolute) |
| `SML_ProfileVelocity` | velocity, PV mode (≈ MC_MoveVelocity) |
| `SML_ProfileVelocity_Jog` | jog (≈ MC_Jog) |
| `SML_Stop` | Halt (≈ MC_Halt) |
| `SML_TouchProbe` | edge-triggered position latch |
| `SML_Status` | decoder of all CiA402 bits |
| `SML_Diagnostics` | aggregates errors → DiagCode/DiagText + first fault |
| `FB_S7RTT_OTG` | jerk-limited trajectory generator (CSP) |

### Orchestration
| File | Role |
|---|---|
| `GVL_App` | application configuration: `MAX_AXIS` (number of axes) |
| `GVL_SML_CONST` | library constants: `PROGRESS_SPAN`, `MAP_SIZE_*` |
| `GVL_AXIS` | array `[1..MAX_AXIS]` of Axis/Ctrl/Data/State/Info + `Control` (FB) + `ItfSmlAxis` |
| `MAIN` | runs all axes every cycle; calls the I/O bridge and MAPPING |

### I/O toward the drive (see `GUIDA_IO_Linking.md`)
| File | Role |
|---|---|
| `OpenSML_Axis` | the axis's CiA402 PDO image (ControlWord/StatusWord/…); `GVL_AXIS.Axis[n]` |
| `ST_DriveOut` / `ST_DriveIn` | output / input half of OpenSML_Axis, for the bridge |
| `GVL_IO` | `AT %Q*/%I*` images + `IO_LINK_ENABLE` flag |
| `FB_IoLink_In` / `_Out` | drive↔struct copy loop (library FBs, instantiated in MAIN) |

### Bus MAPPING (expose Ctrl/State over ADS/fieldbus — optional)
| File | Role |
|---|---|
| `U_AXIS_CTRL` / `U_MOVE_DATA` / `U_AXIS_STATE` / `U_AXIS_INFO` | struct/raw UNION |
| `GVL_AXIS_MAP` | bus-side UNION array + `AXIS_MAP_ENABLE` flag |
| `FB_Mapping_In` / `FB_Mapping_Out` | bus↔internal copy (library FBs, gated by xEnable) |

### Utility / Test
| File | Role |
|---|---|
| `f_GetProgress` / `f_GetState` | decompose the combined state |
| `PRG_LevelA_Test` / `PRG_LevelB_Test` / `PRG_MultiAxis_Test` | simulation benches |

---

## 9. Typical sequences

### 9.1 Enable → homing → move (with status waits)
```pascal
CASE iSeq OF
  0: GVL_AXIS.Ctrl[1].eCmd := AXIS_ENABLE;
     IF GVL_AXIS.Info[1].xEnabled THEN iSeq := 10; END_IF
 10: GVL_AXIS.Ctrl[1].eCmd := AXIS_HOME;
     IF GVL_AXIS.State[1].xDone THEN iSeq := 20; END_IF
 20: GVL_AXIS.Data[1].lrTargetPosition := 50.0;
     GVL_AXIS.Data[1].lrVelocity       := 100.0;
     GVL_AXIS.Ctrl[1].eCmd := AXIS_MOVE_ABS;
     IF GVL_AXIS.State[1].xDone THEN iSeq := 30; END_IF
 30: GVL_AXIS.Ctrl[1].eCmd := AXIS_NULL;  // stopped, enabled
END_CASE
```

### 9.2 Error handling
```pascal
IF GVL_AXIS.State[1].xError THEN
    // read GVL_AXIS.State[1].DiagText / FirstFaultText
    GVL_AXIS.Ctrl[1].eCmd := AXIS_RESET;      // clear the fault
    // then re-enable: AXIS_ENABLE
END_IF
```

---

## 10. Integrating into the project

1. **Import** the files (order and steps in `IMPORT_CHECKLIST.md`). Remember to create the
   `I_Axis` METHODs/PROPERTYs under `FB_AxisCtrl` (`FB_AxisCtrl_METHODS`).
2. **Number of axes**: set `GVL_App.MAX_AXIS`.
3. **Task**: put **`MAIN`** in a cyclic task. (In simulation, use a test bench INSTEAD of MAIN.)
4. **I/O**: link `GVL_AXIS.Axis[n]` to the drive PDOs — see `GUIDA_IO_Linking.md`
   (CoDeSys: direct link; TwinCAT: bridge + `IO_LINK_ENABLE := TRUE`).
5. **Application**: write `GVL_AXIS.Ctrl[n]`/`Data[n]`, read `State[n]`/`Info[n]`.

---

## 11. Simulation (without a drive)

Set `Ctrl.xSimulation := TRUE` (mirrors position). The benches
`PRG_LevelB_Test` / `PRG_MultiAxis_Test` include a mini CiA402 emulator and exercise the command
sequence: put ONE bench in the task (not MAIN) and check `xTestPassed = TRUE`.

---

## 12. Diagnostics (`SML_DiagCode`)

Read `State.DiagCode`/`DiagText` (live condition) and
`State.FirstFaultCode`/`FirstFaultText` (first fault). Categories:
- `1..9` warning (motion continues) — e.g. `DIAG_INTERNAL_LIMIT`
- `10..19` drive fault — e.g. `DIAG_DRIVE_FAULT`, `DIAG_FOLLOWING_ERROR`, `DIAG_QUICK_STOP`
- `20..29` FB fault — e.g. `DIAG_HOME_ERROR`, `DIAG_MOVE_ERROR`, `DIAG_OTG_ERROR`, `DIAG_TOUCHPROBE_ERROR`

Reset: `AXIS_RESET` (clears the fault of the drive and of the leaf FBs).

---

## 13. Common mistakes (gotchas)

- **Won't start / won't enable** → `Ctrl.xEmergencyStop` must be **TRUE** (TRUE = run). Check
  `Info.Status.Fault`/`DiagText`.
- **Moves to the wrong position** → wrong `Ctrl.Scale` (counts/unit).
- **CSP doesn't follow / strange trajectory** → `CycleTime` ≠ the real task period.
- **Second PP move doesn't start** → it's discrete: re-trigger (go through `AXIS_NULL`) or use
  `AXIS_MOVE_CSP` for online re-targeting.
- **In-position never TRUE (CSP)** → `PositionWindow` too tight.
- **Touch-probe doesn't latch** → PDO 0x60B8/9/A/B not mapped, or DC not active.

---

## 14. Quick glossary

- **PP / PV / CSP**: Profile Position / Profile Velocity / Cyclic Synchronous Position (mode 8, with OTG).
- **OTG**: online trajectory generator (jerk-limited) = `FB_S7RTT_OTG`.
- **User units**: yours (mm, revs…); the layer converts to counts with `Scale`.
- **Combined state**: `eState = functional state + progress`.

---

## Appendix A — Example application FB (`FB_AxisCycleDemo`)

A complete, importable example (`FB_AxisCycleDemo.txt`, **not** part of the library) that shows the
real usage pattern: a state machine that commands an axis in a **continuous A↔B cycle** with enable,
one-time homing, dwells, stop and automatic reset on error.

### A.1 What it does
```
ENABLE → HOME (if not already homed) → A → dwell → B → dwell → A → … (until xStop)
error at any point → automatic RESET → idle (needs a new xStart)
```

### A.2 Interface
| Input | Type | Use |
|---|---|---|
| `nAxis` | UINT | axis index in `GVL_AXIS` [1..MAX_AXIS] |
| `xStart` | BOOL | start the cycle |
| `xStop` | BOOL | stop (Halt) and close the cycle |
| `lrPosA` / `lrPosB` | LREAL | the two positions [units] |
| `lrVel` | LREAL | move velocity [units/s] |
| `tDwell` | TIME | dwell at each end |

| Output | Type | Use |
|---|---|---|
| `xBusy` / `xDone` / `xError` | BOOL | cycle status |
| `iStep` | INT | current step (for debug) |
| `sDiag` | STRING | axis diagnostic text |

### A.3 The two techniques demonstrated
1. **Command → wait for result**: each step sets `Ctrl.eCmd` (+ `Data`) and waits for
   `GVL_AXIS.State[nAxis].xDone` before continuing.
2. **Re-triggering discrete PP moves**: between `A` and `B` it goes through a step with `AXIS_NULL`
   (the dwell), which creates the **edge** needed to restart the next move. (For smooth re-targeting
   you would use `AXIS_MOVE_CSP`.)
3. **One-time homing**: skips homing if `Info.xHomed` is already TRUE.
4. **Error handling**: the supervision at the top of the FB catches `State.xError` and jumps to the
   RESET step.

### A.4 How to use it
```pascal
PROGRAM PLC_APP
VAR
    cycleAx1 : FB_AxisCycleDemo;
    xStartBtn, xStopBtn : BOOL;   // from HMI / buttons
END_VAR

cycleAx1(
    nAxis  := 1,
    xStart := xStartBtn,
    xStop  := xStopBtn,
    lrPosA := 0.0,
    lrPosB := 250.0,
    lrVel  := 150.0,
    tDwell := T#1S);
// readable: cycleAx1.xBusy, cycleAx1.xError, cycleAx1.iStep, cycleAx1.sDiag
```
Put it in a task **together with `MAIN`** (which runs the axes). For two axes in an independent cycle:
two instances with `nAxis := 1` and `nAxis := 2`.

### A.5 Try it in simulation
Set `GVL_AXIS.Ctrl[1].xSimulation := TRUE` (or use a bench with the emulator), put `MAIN` + `PLC_APP`
in the task, force `xStartBtn := TRUE` and watch `iStep` advance 10→20→30→40→50→60→30… and
`Info[1].lrActualPosition` oscillate between A and B.

---

## Appendix B — Complete 2-axis machine (`PLC_APP`)

`PLC_APP.txt` is a **realistic**, importable example (not part of the library): piece
measuring/sorting on a belt. Axis 1 = continuous belt (velocity + touch-probe), Axis 2 = positioner.
It demonstrates: two axes, touch-probe, velocity→PP transition without stopping, a decision on the
measurement, simultaneous motions, clean machine I/O.

### B.1 Inputs / Outputs (integration)
| Input (`PLC_APP.`) | Use |
|---|---|
| `xStart` | start/hold the automatic cycle |
| `xStop` | stop (Halt) and go to idle |
| `xSafetyOk` | safety consent (TRUE = run; → `xEmergencyStop`) |
| `xPhotocell` | measure photocell, TRUE = covered by a piece |
| `xExitPhotocell` | exit photocell, TRUE = rejected piece passing through |
| `xResume` | external signal: reactivate the belt from standby |
| `xHomeBelt` | re-home the belt while it is stopped in standby |
| `lrScale, lrCycleTime, lrBeltVel, lrBeltMove, lrThreshold, lrAx2Home, lrAx2Fwd, lrAx2Zero, lrAx2Vel, tWait, tIdle` | parameters (example defaults) |

| Output (`PLC_APP.`) | Use |
|---|---|
| `xHomed` | homing complete |
| `xRunning` | cycle active |
| `xStandby` | belt stopped in standby (no pieces) |
| `xError` / `sDiag` | axis error + text |
| `xOversize` | last piece out of tolerance |
| `xEjected` | pulse: certified reject ejected (exit photocell) |
| `nRejectCount` | ejected-reject counter |
| `lrLastMeasure` | last piece measurement [mm] |
| `iStep` | current step (debug) |

### B.2 Step map (`iStep`)
```
 0 idle → 5 enable(both axes) → [10 home belt → 20 home axis 2] (once)
   ↓
 30 belt VELOCITY + arm touch-probe
 40 wait for photocell COVERED → belt switches to relative PP (lrBeltMove)
    │  (if no piece for tIdle → 100 STANDBY)
 50 wait for photocell CLEARED → measurement = |TP_falling − TP_rising|/Scale
      ├─ measurement > threshold → belt stays in VELOCITY, go back to 30 IMMEDIATELY
      │                             (reject ejected and counted in PARALLEL, pipelining)
      └─ measurement ≤ threshold → 70 wait for belt stopped → 75 wait+axis 2 forward to +400
                            → 80 wait for axis 2 → 85 wait+(axis 2→0 ∥ belt velocity) → 30

 100 STANDBY: belt decelerates and stops → 110 stopped
 110 waits for xHomeBelt (→120 re-home) or xResume (→30 restart)
 120 home belt in place → 110

 PARALLEL (always): falling edge of xExitPhotocell → nRejectCount++, xEjected
200 STOP (Halt both axes) → idle     900 ERROR (reset both axes) → idle
```

### B.3 Usage
```pascal
// in the task, TOGETHER with MAIN:
PLC_APP.xSafetyOk      := xSafetyConsent;
PLC_APP.xPhotocell     := xMeasurePhotocell;
PLC_APP.xExitPhotocell := xExitPhotocell;
PLC_APP.xStart         := xStartMachine;
PLC_APP.xStop          := xStopMachine;
PLC_APP.xResume        := xResumeFromStandby;   // external signal
PLC_APP.xHomeBelt      := xHomeBelt;            // in standby
// (opt.) PLC_APP.lrThreshold := 300.0;  PLC_APP.tIdle := T#5S;  etc.
// read: PLC_APP.xRunning, PLC_APP.xStandby, PLC_APP.lrLastMeasure,
//       PLC_APP.xOversize, PLC_APP.nRejectCount, PLC_APP.iStep
```

### B.4 Drive requirements (important)
- **Measure photocell** wired to the **touch-probe trigger** of axis 1's drive (for the rising/falling
  latches) **and** to the PLC input `xPhotocell` (for sequencing).
- **Exit photocell** (`xExitPhotocell`) downstream: certifies that the rejected piece has passed (no
  actuator: the piece falls off the belt).
- **Axis 1 homing**: "set position" method (in place).
- **Axis 2 homing**: sensor-in-negative + encoder index method; the reference becomes `lrAx2Home`
  thanks to `HomeOffset` (set by the app in counts). The sensor/index is handled by the drive's homing
  method.
- **DC (Distributed Clocks)** active for the touch-probe.
- Rotary belt: if the encoder **wraps** (modulo), the `falling − rising` measurement must be handled
  with modulo arithmetic (not covered here).

### B.5 Assumptions (adaptable)
Belt PP travel = **relative** from the trigger (`lrBeltMove`); axis 2 "forward" = **absolute**
`lrAx2Fwd` (+400), "return" = **absolute** `lrAx2Zero` (0); reject = belt keeps running in velocity +
certified on `xExitPhotocell` (`nRejectCount`). If your layout differs, change the parameters or let me
know and adapt the logic.
